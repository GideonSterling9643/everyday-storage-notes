# Private Object Storage URL Governance Explained — 4 Controls for Signed Gaming Downloads

Short answer: for signed gaming documents, use private object storage behind a small application-owned download service, isolate tenants in the object namespace and authorization policy, issue short-lived signed URLs only after an entitlement check, and enforce the document's deletion deadline with a separate, auditable deletion path. An expiring URL is a credential deadline, not a data-retention control.

This is an architecture decision record for a game publisher that lets studio tenants export signed tournament agreements and consent records. The application serves users in the US and EU, but tenant isolation is the primary decision axis: a valid link for tenant A must never identify or authorize an object owned by tenant B. S3 API compatibility helps with portability, yet it doesn't prove identical policy, consistency, residency, lifecycle, or signing behavior. Those claims need tests.

The decision is intentionally narrow. Keep objects private, let the application decide who may download, and make the storage layer responsible for bytes rather than business identity. Four invariants govern the design:

1. Every object has one immutable tenant owner and a non-guessable object identifier.
2. The signing service derives the object key from trusted metadata, never from a client-supplied bucket or key.
3. URL expiry is measured in minutes; the explicit deletion deadline is stored independently and enforced even if nobody requests a download.
4. A download response uses a safe filename while logs, metrics, and object keys avoid player names, email addresses, and contract titles.

Miss any one of these and a successful download can still be the wrong outcome.

## What should an evaluation test for private SaaS file exports and signed download URLs?

Start with two boundaries, because they fail differently. The control plane contains tenant membership, document status, region assignment, the deletion deadline, and the right to mint a temporary link. The data plane stores and returns opaque bytes. A request such as `POST /documents/{document_id}/download` belongs to the application control plane: authenticate the user, load the document by both `tenant_id` and `document_id`, reject a deleted or unauthorized record, then ask the storage adapter to sign the already-resolved key. The browser receives a temporary URL, follows it, and downloads directly from the object service.

The crucial query includes the tenant. Looking up a document by a globally unique identifier and checking ownership later creates a dangerous gap: future refactors can skip the second check, error responses can reveal that another tenant's document exists, and a cache keyed only by document ID can cross the boundary. Make `(tenant_id, document_id)` the lookup key all the way through the authorization transaction. The storage key should repeat that boundary, for example `tenants/<opaque-tenant-id>/signed-records/<opaque-document-id>/<version>`, even when each tenant already has a separate bucket. Defense in depth is useful here because operational tooling often has broader access than the application.

Don't put a signed URL in an analytics event, support ticket, referrer-bearing page, or normal application log. It is a bearer credential until it expires. Redact its query string at ingress and egress, mark the download response to discourage caching where appropriate, and keep the lifetime only as long as a normal transfer needs. For very large exports, that duration must account for when the object service validates expiry; I'm not sure any claimed behavior is safe for a chosen provider until an integration test starts a transfer near expiry and records the result.

`Content-Disposition` deserves explicit handling. An `attachment` disposition asks the user agent to download the response, while `filename` and `filename*` influence the suggested name; MDN also documents browser differences and the encoding considerations around those parameters. Generate a conservative ASCII fallback, add a correctly encoded UTF-8 name when needed, strip control characters and path separators, and never let an uploaded filename become a raw response-header fragment.

## Roll out deletion enforcement before link issuance

A signed-link feature usually looks complete in a happy-path demo: upload a PDF, sign a request, click it. The production failure modes sit around that path.

| Boundary | Failure to test | Required evidence |
| --- | --- | --- |
| Tenant authorization | A user from tenant A asks for tenant B's document ID | Identical denial behavior without an object lookup or existence leak |
| Region placement | An EU tenant is mapped to a US bucket during retry or migration | Region recorded in authoritative metadata and checked before upload and signing |
| Retention | URL expiry is mistaken for object deletion | Deletion job, tombstone, retry state, and periodic inventory reconciliation |
| Naming | A document title injects a header or exposes personal data in logs | Filename sanitizer tests and query-string redaction tests |
| Versioning | A replacement upload leaves an old signed record reachable | Version-aware metadata plus deletion rules for every retained version |
| Clock and expiry | Signer, verifier, or client clocks disagree | Bounded skew tests and conservative URL lifetime |
| Large transfer | A link expires near the start or middle of a download | Provider-specific integration test using a throttled transfer |

Deletion is the boundary most teams underestimate. Suppose a consent record is created at 14:03 UTC with `delete_after` set to 30 days later. The application can issue a five-minute URL today, but the object remains present after those five minutes. At the deadline, a worker should claim the due record idempotently, delete the exact object version or all versions required by policy, verify the expected absence through the adapter's documented semantics, and write an audit event that contains opaque identifiers rather than personal data. If deletion fails for a transient client-side or network reason, the record remains due and retries with bounded backoff; the system must not mark it deleted merely because a delete request was attempted. A separate reconciliation pass compares authoritative metadata with storage inventory, because queues can lose work and humans can create objects outside the normal path.

Short links aren't deletion.

Retention policy also has a legitimate conflict with legal hold. Represent that conflict as state, not as a comment or an exceptionally long URL lifetime: `delete_after`, `hold_status`, `hold_reason_code`, and a restricted audit trail let the deletion worker make a deterministic decision. The exact evidence period and regional rules are legal inputs, not storage defaults, so an architect should demand them before choosing lifecycle intervals.

## Reliability starts with the tenant isolation model

The provider shortlist comes later. First choose the blast radius the organization can operate.

| Model | Isolation strength | Operational cost | Best fit | Main limitation |
| --- | --- | --- | --- | --- |
| Shared bucket, tenant key prefixes | Depends heavily on application and policy correctness | Lowest bucket and policy count | Many small tenants with uniform controls | One broad credential or policy error can expose a large population |
| Bucket per tenant | Stronger policy and operational boundary | More buckets, policies, lifecycle rules, and inventory work | Fewer high-value tenants or contractual isolation needs | Quotas and control-plane automation become design constraints |
| Account or project per tenant | Strongest administrative and billing boundary of these options | Highest provisioning and support burden | Regulated or unusually large tenants | Not suitable for a high-churn, self-service tenant base |
| Bucket per region, prefixes per tenant | Clear residency boundary with moderate object layout complexity | Requires a trustworthy tenant-to-region mapping | US/EU service with uniform controls inside each region | Region reassignment and cross-region recovery need explicit procedures |

For this gaming system, bucket per region with tenant prefixes is the default, paired with tenant-scoped metadata queries and a signer that has access only to the assigned region. The catch is the shared failure domain: it is not suitable when a studio contract requires a dedicated administrative boundary or customer-managed keys. Use bucket-per-tenant for those studios if the chosen service's quotas and policy automation have been proven at the projected tenant count. Use separate accounts or projects when administrators, billing, and incident response must also be isolated, accepting slower provisioning and more operational machinery.

AWS S3, Cloudflare R2, and Backblaze B2 are reasonable names to include in a test matrix because teams commonly encounter them while evaluating S3-compatible storage; inclusion is not a ranking. Treat each as its own target. Run the same conformance suite against the exact region, endpoint, signing library, and account configuration you intend to deploy, and record results for conditional requests, multipart uploads, range downloads, metadata preservation, lifecycle behavior, version deletion, and URL expiry. Backblaze publishes its current storage, download, and transaction charging model on its pricing page, which is a reminder to model request and egress patterns rather than compare one headline storage number. Prices and policies can change, so the reviewed pricing page and test date belong in the decision record.

## Record the adapter boundary as an architecture decision

The web application should not accept raw object coordinates from the browser. The example below is deliberately a storage-neutral Python contract even if the calling SaaS application is written in Node.js; the important artifact is the boundary, and every production implementation needs the same authorization and retention behavior. Application code sees only an opaque document record and a constrained download request.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Protocol


@dataclass(frozen=True)
class Document:
    tenant_id: str
    document_id: str
    region: str
    object_key: str
    download_name: str
    delete_after: datetime
    deleted_at: datetime | None


class ObjectStore(Protocol):
    def sign_download(
        self,
        *,
        region: str,
        object_key: str,
        expires_in: timedelta,
        download_name: str,
    ) -> str: ...


def issue_download(
    *,
    tenant_id: str,
    document_id: str,
    repository,
    store: ObjectStore,
    now: datetime,
) -> dict[str, str | int]:
    # This query must enforce the tenant boundary in the data store.
    document = repository.get_for_tenant(
        tenant_id=tenant_id,
        document_id=document_id,
    )
    if document is None or document.deleted_at is not None:
        raise PermissionError("download unavailable")
    if now >= document.delete_after:
        raise PermissionError("download unavailable")

    lifetime = timedelta(minutes=5)
    remaining = document.delete_after - now
    effective_lifetime = min(lifetime, remaining)

    url = store.sign_download(
        region=document.region,
        object_key=document.object_key,
        expires_in=effective_lifetime,
        download_name=document.download_name,
    )
    return {"url": url, "expires_in_seconds": int(effective_lifetime.total_seconds())}


now = datetime.now(timezone.utc)
```

Three details matter. The repository method binds tenant and document in one query. The response uses one generic denial so callers cannot distinguish a missing object from another tenant's object. Finally, the URL cannot outlive the deletion deadline, although object deletion still belongs to the independent retention worker.

Deployment should fail closed. Give the signer read-and-sign capability for the intended bucket scope, give the uploader write capability without broad read access, and give the deletion worker only the delete scope it needs. Rotate credentials, but don't assume rotation revokes an already issued URL; test revocation expectations explicitly. Metrics should count signing outcomes, deletion lag, overdue objects, reconciliation mismatches, and downloads by tenant and region without recording URLs or filenames. Alert on overdue deletion and cross-region routing attempts, not just request latency.

## Compare direct delivery with the proxy exception

Streaming every export through the SaaS application can centralize authorization and immediate revocation, and it may be the correct choice when each byte must pass through content transformation, watermarking, or a policy enforcement point. It also makes the application responsible for long-lived connections, range requests, backpressure, bandwidth capacity, and partial-transfer behavior. For ordinary immutable signed records, those responsibilities obscure the authorization boundary and expand the service's failure surface, so direct temporary downloads are the recorded decision.

The rejected option remains valid when access must be revoked during an active session, when the storage service cannot express required response headers, or when a regulator requires synchronous inspection on every read. Stick with the proxy in those cases and capacity-test it as a data plane, not as a normal JSON endpoint.

Whichever path is chosen, acceptance is concrete: tenant-crossing tests always deny; filenames cannot inject headers; a five-minute credential never authorizes past its limit; due objects and retained versions are deleted or held with an auditable reason; US and EU mappings cannot silently cross; and reconciliation detects objects that bypassed the application. S3 compatibility alone answers none of those questions.

## References

- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [Backblaze B2 pricing](https://www.backblaze.com/cloud-storage/pricing)
