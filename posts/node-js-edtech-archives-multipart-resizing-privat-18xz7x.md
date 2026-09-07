# Node.js Edtech Archives: Multipart Resizing, Private Originals, and Signed Download Links

The decisive trade-off is not which object store has the shortest feature list. It is whether large signed documents can enter the system at full throughput while every original remains private, every derivative is reproducible, and the deletion deadline survives retries. For a Node.js edtech SaaS, choose an object-storage interface that supports multipart upload, private objects, short-lived signed download links, explicit object metadata, and lifecycle or deletion operations you can verify independently. Keep resizing in workers rather than the request path, and keep the authoritative retention record in a database rather than inferring it from a bucket listing.

That design is simple in the useful sense: each component has one source of truth. It isn't the fewest-component design, and the distinction matters once a 2 GB signed course packet arrives while a thumbnail backlog is already draining.

## What fails when Node.js resizes private originals for signed download links?

Multipart support is necessary for large-file throughput, but it is not a throughput guarantee. Measure the entire path: time to initiate, part concurrency, retry volume, completion latency, queue age, worker download time, transform time, derivative upload time, and time until the document becomes readable. A fast bucket behind an undersized resize pool is still a slow product.

Backpressure belongs at each handoff. Limit concurrent parts per upload, cap active transform work by memory rather than CPU count alone, and separate interactive previews from bulk reprocessing. Image decoders can expand a compact input into a much larger in-memory representation, so admission should use decoded dimensions and the worker's measured memory envelope. The exact ceiling depends on codecs, libraries, and deployment limits; I'm not sure a static rule is defensible without workload measurements. Record peak resident memory during representative transformations and set the queue's worker concurrency from that evidence.

Small requests need a different lane. A user waiting for the first page preview should not sit behind a batch that regenerates every thumbnail after a rendering change.

Multipart sizing also creates a balancing act. Very small parts increase request and bookkeeping overhead; very large parts make each retry expensive and reduce useful parallelism. Use a load test containing the actual document-size distribution, inject failed part attempts, and compare both throughput and abandoned-upload cleanup. The result should be a range, not a magic universal number.

The storage adapter needs only a narrow contract: begin an upload, upload a numbered part, complete or abort it, put and read an object, issue a time-bounded download capability, and delete a named object. Keep provider response shapes below that boundary. If changing storage means editing authorization or retention logic, the abstraction has leaked.

**Load-test the two clocks before choosing storage.**

Start with three object classes, even if they share one physical storage account. An `original` is immutable input, a `preview` is a derived image safe for the intended viewer, and a `manifest` records the relationship between them. The application database owns authorization, document state, and `delete_after`; object storage owns bytes. A signed link is a temporary delivery capability, not evidence that the user is still entitled to the document. The upload path should issue a stable document identifier before transferring bytes. Large files then use multipart upload, which divides an object into independently uploaded parts and completes the object only after those parts have arrived. The multipart overview in the Sources section also calls out the operational obligation people skip: uploaded parts continue to consume storage until the upload is completed or aborted. Track the upload identifier and an expiry time in durable state so an abort sweeper can clean abandoned sessions. Don't resize during the web request. Completion of the original should enqueue an idempotent derivative job keyed by the original's immutable identity plus a transformation version, for example `document_id/original_version/preview_v3`. The worker downloads the original, validates the expected identity, writes the preview, and atomically records that preview as ready. A retry writes the same logical derivative; it must not create a new lineage every time. There is a hard boundary here. The browser may upload or download through a signed URL, but it should never receive credentials that can list or delete objects. Cross-origin browser calls also need an explicit CORS policy: allowed origins, methods, and headers are access rules enforced by the browser, while application authorization remains a separate decision. A permissive CORS response doesn't make an otherwise unauthorized request legitimate, and a strict CORS response doesn't replace signed-link validation.

For signed documents, the database row can move through `uploading`, `processing`, `ready`, `deleting`, and `deleted`. Readers are admitted only in `ready`. Deletion begins by denying new links, then removes every object named in the manifest, and finally records completion. That ordering closes the most embarrassing gap: issuing a fresh download capability after the legal or contractual deadline has arrived.

## Implement deletion as a scheduled state transition

An expiry rule is useful defense in depth, but a signed-document system needs evidence tied to the application record. `delete_after` should be written when the retention decision is made and should not be quietly extended by a thumbnail retry, metadata update, or download. At the deadline, deny new links first. Then delete the original, all known derivatives, and the manifest using explicit keys from durable state.

Deletion must be idempotent because workers retry. The compact Python model below is intentionally storage-neutral; `store.delete()` is expected to treat an already absent named object as the desired end state, while the database transaction prevents a half-finished attempt from looking complete.

```python
from dataclasses import dataclass
from datetime import datetime, timezone


@dataclass(frozen=True)
class RetainedDocument:
    document_id: str
    delete_after: datetime
    object_keys: tuple[str, ...]
    state: str


def delete_due_document(document, store, repository, now=None):
    now = now or datetime.now(timezone.utc)
    if document.state == "deleted" or now < document.delete_after:
        return False

    repository.block_new_downloads(document.document_id)
    repository.mark_deleting(document.document_id)

    for object_key in document.object_keys:
        store.delete(object_key)

    repository.mark_deleted(document.document_id, deleted_at=now)
    return True
```

The manifest must be complete before `ready`, or deletion cannot prove what it covered. Avoid discovering derivatives from a prefix at deletion time: naming bugs, version changes, or an interrupted write can make a listing disagree with application intent. A reconciliation job can compare the manifest against storage and raise an alert for unknown or missing keys, but reconciliation should not invent authorization state.

Watch the awkward intervals — they define the system. What happens when the deadline arrives during multipart upload? Abort the upload, deny completion in application state, and record the terminal result. What happens when deletion races a preview worker? The worker must re-check document state before publishing the derivative, and the delete workflow must be able to run again against the stable key set. What happens when a signed link outlives the deadline? Prevent that at issuance by capping its expiry at `delete_after`; links already issued are one reason the delivery lifetime should be deliberately short.

Audit events should identify the document, transition, object key or manifest version, attempt, and timestamp without logging a signed URL. Alert on overdue documents, age of the oldest abandoned multipart upload, deletion retries, derivative queue age, and objects that fail reconciliation. Count bytes as well as objects; one stranded original can matter more than thousands of thumbnails.

**Choose the operating boundary from measured constraints.**

Only now is a storage comparison useful. Test candidates against the same workload and contract, then retain the raw observations. Published feature matrices rarely reveal the behavior of a browser upload through your network path or the effort required to demonstrate deletion.

| Operating model | Large-file path | Resizing path | Main limitation | Prefer it when |
| --- | --- | --- | --- | --- |
| Managed object storage plus workers | Multipart upload feeds independently scaled consumers | Your queue and image workers own transformation versions | More application orchestration and reconciliation | Retention evidence, portability, and workload control dominate |
| Storage with an integrated image pipeline | Original upload and derivative delivery share a managed surface | Transformations are configured near delivery | Transformation semantics and migration are more coupled to the service | The supported image operations match the workload and documents do not require specialized processing |
| Self-hosted object storage plus workers | Throughput depends on the cluster, disks, and network you operate | Your workers own every transform | Your team owns capacity, durability operations, upgrades, and incident response | Data placement or operational control outweighs that burden |

The first model is not suitable when the team cannot operate queues, retry policy, and deletion reconciliation; an integrated pipeline may be the better boundary then. Stick with self-hosting when a placement constraint genuinely requires it and the organization already has storage operations expertise. Conversely, don't choose self-hosting merely to make the API look familiar. Familiar calls don't operate disks.

Price can be compared only after recording request volume, stored original and derivative bytes, retention duration, transfer paths, multipart retries, transformation work, and deletion/listing operations. There is no defensible “cheapest” answer without that workload. A low storage rate can be irrelevant if derivatives multiply retained bytes or delivery dominates the bill, while a bundled transformation service can be poor value when most uploaded documents never need a preview.

The acceptance test should include private-by-default objects, rejected unauthorized reads, bounded signed-link expiry, exact multipart completion, abort cleanup, idempotent transforms, deadline races, repeatable deletion, CORS behavior for every supported browser origin, and reconciliation after injected worker termination. Capture p50 and tail latency, but also capture correctness counts: leaked parts, unexpected keys, missing manifests, and overdue deletions. Zero is the target for the last four.

## Migrate by document size band

Begin with shadow writes of manifests and state transitions while the current delivery path remains authoritative. Next, enable multipart upload for an internal cohort, run derivative workers in parallel, and compare object identity, preview output, queue latency, and deletion results. Expand by document class and size band, not by an arbitrary percentage that mixes easy thumbnails with the largest signed packets.

Keep rollback asymmetric: stop issuing new uploads to the candidate path, but continue draining its queued transforms and deletion obligations. Never strand retention duties during a rollback. Before broad release, restore a representative backup into an isolated environment, exercise authorization and CORS from the real application origin, expire links, and run the deadline workflow twice to prove idempotency.

The final decision rule is deliberately plain. Select the operating model that meets measured large-file throughput while preserving private originals, reproducible derivatives, bounded download capabilities, and auditable deletion under retry. Everything else is negotiable.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
