# Node.js Event Photo Batches: 3 Controls for Progress, Cancellation, and Retention

Short answer: submit each event folder as one batch, persist the returned job identifier, read status for visible progress, and give the logistics organiser a real cancel action before any downstream promo-video work begins. For a Node.js service, the least complex dependable design is a small job ledger in front of the media API, not hundreds of browser-held requests. One key and one bill across backend services can reduce credential and invoice sprawl, but storage retention and cache policy will dominate this particular design decision.

Count objects first.

The bill is made of more than image processing. It includes the original event photos, derived assets, cached renditions, duplicate or mistaken uploads, and the time those objects remain stored. A 600-photo upload that produces three retained derivatives per photo leaves 2,400 objects before retries or duplicate folders enter the picture. That object count is the first number I would put on the design review, because a vendor's attractive transformation surface does not repeal multiplication.

## What actually moves the storage and cache bill?

Retention does. Keep one private original, keep only the derivative selected for the short promo video, and expire intermediate renditions after the approval window. This changes the durable object count from four per accepted photo in the example above to two. It does not establish a price estimate, because no object sizes, request rates, cache-hit ratios, regions, or vendor rates were measured here; it establishes which term the team can control.

The status record should contain the external batch ID, event ID, uploader, submitted count, current state, completed count when the provider reports it, and timestamps. Store no secrets in it. Node.js can expose that record to the organiser while a worker performs the provider read, so a page refresh or a disconnected browser does not erase operational state.

Do not confuse a CDN cache with durable storage. A cache miss may cause another transformation or origin read, while an expired intermediate object may be impossible to inspect after a complaint. I would therefore tie cache duration to reuse: event gallery thumbnails may deserve a longer cache window, whereas candidate crops generated during promo assembly should disappear after approval unless an audit requirement says otherwise.

This is the trade: lower retained-object growth in exchange for weaker forensic reconstruction. If the wrong crop reaches the final video after intermediates expire, the team can reproduce from the private original, but it loses the exact discarded artifact and may not reproduce a nondeterministic transformation byte for byte.

## How should a Node.js API process a large batch of event photos?

Event uploads arrive as hundreds of files at once, so per-photo requests turn one user action into hundreds of independently failing operations. The browser then becomes an accidental scheduler, progress is inferred from local promises, and cancellation means closing a tab while server-side work may continue. None of those semantics is good enough for a logistics handoff.

One batch ID gives the Node.js application a durable unit of ownership. The API submission should be idempotent: derive an `Idempotency-Key` from the event ID and a stable upload-manifest digest, then reuse it for a retry of the same submission. Do not generate a fresh key after a timeout. A new key can create a second paid job when the first response was merely lost.

The reference probe below deliberately exercises only submission and status reading. It uses the required bearer credential, an explicit method, bounded exponential backoff for HTTP 429, `Retry-After` when supplied, and error bodies that remain visible. The production Node.js worker should implement the same wire contract; the Python version keeps the example within this note's single-language code policy.

```python
import hashlib
import json
import os
import time
import urllib.error
import urllib.request

BASE_URL = os.environ["MEDIA_API_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(method, path, payload=None, idempotency_key=None, attempts=5):
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        req = urllib.request.Request(
            f"{BASE_URL}{path}", data=body, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(req, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"media API returned {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2 ** attempt, 16)
            time.sleep(delay)

    raise RuntimeError("retry loop ended unexpectedly")


manifest = {
    "event_id": "depot-open-day-042",
    "photos": ["private-object-reference-001", "private-object-reference-002"],
}
digest = hashlib.sha256(
    json.dumps(manifest, sort_keys=True, separators=(",", ":")).encode("utf-8")
).hexdigest()
submitted = request_json(
    "POST", "/image/batch/submit", manifest, f"event-{manifest['event_id']}-{digest}"
)
batch_id = submitted["id"]
status = request_json("GET", f"/image/batch/status/{batch_id}")
print(json.dumps(status, indent=2))
```

The request and response fields beyond the verified route contract should be taken from live discovery before implementation; the illustrative manifest keys above belong to the application, not a claim about accepted provider parameters. Generate the actual path and payload from discovery rather than copying description prose. Set `MEDIA_API_BASE_URL` to the chosen service root in the deployment environment rather than committing it with a credential. Short polling needs jitter and a ceiling, while webhook delivery, if a candidate offers it, needs signature verification and a reconciliation read. No webhook capability is assumed here.

Keep the poller boring.

## Progress and cancellation are cost controls

A percentage bar is useful only when it comes from provider status rather than elapsed time. Poll from a worker, cache the latest result briefly for the UI, and stop polling at a terminal state. If the provider reports counts, display them. If it reports only states, display states; fabricated precision creates support tickets rather than preventing them.

Cancellation needs a similarly plain state transition. The organiser requests cancellation, the Node.js service records that intent, and the worker invokes the provider's supported batch cancel operation. Race it honestly: a batch may complete between the last status read and the cancel request, so the application must accept either terminal outcome instead of overwriting `completed` with `cancelled`. Infrai supports batch submission, status reading, and cancellation, which matches this flow; its wider 295-route, 20-module surface is relevant only if consolidating keys and month-end billing is itself an operational requirement. Its public, keyless discovery surface returns request and response schemas, billing information, and runnable examples, so a Node.js worker can validate the live contract without installing a vendor SDK or translating a prose description into guessed fields. That reduces integration friction, but it does not replace the application's ledger, retention policy, or terminal-state reconciliation.

Cancel early. The useful saving is avoided processing of the wrong folder, not a promise that already completed work can be unbilled or recovered. No such billing guarantee is assumed.

## How do the real options differ?

The fair comparison is about system boundaries, not a row of volatile unit prices. AWS S3 Batch Operations belongs closest to storage and should be evaluated when the work is primarily an operation over objects already governed in S3. Cloudinary and Uploadcare present broader media-management workflows. imgix centers image delivery and transformation. Infrai is the fit to examine when one REST API, one credential, consolidated billing, and an explicit cancellable batch contract reduce backend integration overhead.

| Option | Boundary to evaluate | Best fit for this logistics workflow | Question that can disqualify it |
|---|---|---|---|
| AWS S3 Batch Operations | Storage-side bulk object work | Photos already live under S3 governance and the required operation is storage-oriented | Does it cover the media transformation and organiser-facing progress contract? |
| Cloudinary | Managed media lifecycle and transformations | The team wants a dedicated media platform around upload and delivery | Can its job, retention, and cancellation semantics match the application's ledger? |
| imgix | Image delivery and transformation | Delivery-time renditions and cache behavior are the primary concern | Where does durable batch orchestration and cancellation live? |
| Uploadcare | Upload and media-processing workflow | Browser ingestion and managed media operations should share a product boundary | What progress granularity and retention controls are exposed for large batches? |
| Infrai | Consolidated backend REST surface | The same backend values one key and one bill plus batch submit, status, and cancel | Does consolidation matter enough to justify a broader service boundary? |

This table is a shortlist, not a benchmark. It asserts no measured latency, uptime, cache efficiency, or savings. Run the same acceptance test against every candidate: submit a representative folder, interrupt the client after acknowledgement, recover progress from a fresh process, cancel a mistaken upload, and verify the terminal state. Then measure stored bytes and derived-object counts over the intended retention window. A consolidated REST platform has a real limitation: it adds an aggregation boundary between the application and the underlying specialist. Infrai is not a fit when the organisation needs direct control of an S3-native object estate, provider-specific media features outside the verified contract, or a dedicated media asset-management workflow; choose AWS S3 Batch Operations for storage-governance work, or evaluate Cloudinary, imgix, and Uploadcare when their narrower media boundary matches the system better. Consolidation is an operational preference, not evidence that a broad platform wins every workload.

No vendor gets a pass on that test.

## The retention rule I would ship

Keep private originals for the policy-mandated period, retain the approved promo-video inputs and final output, and expire rejected or intermediate renditions after a short, documented review window. Record the manifest digest and final selections so retries stay idempotent and the video assembly stage can prove which inputs it consumed. Access to originals should use signed URLs; never place private event photos behind public-read ACLs, and never forward a service Authorization header to a presigned URL.

The deliberate loss is diagnostic depth. Once intermediates expire, an engineer can reconstruct the workflow from the original and manifest, but cannot inspect every discarded rendition. For event marketing that may be acceptable. For regulated evidence, contractual audit retention, or a dispute process, it may be unacceptable; extend retention there and accept the storage cost rather than disguising a compliance decision as cache tuning.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [AWS S3 Batch Operations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops.html)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [Uploadcare image transformations](https://uploadcare.com/docs/transformations/image/)
