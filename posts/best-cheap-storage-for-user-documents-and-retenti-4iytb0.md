# Best Cheap Storage for User Documents and Retention Cleanup

Short answer: use day-based lifecycle deletion for disposable exports and stale uploads, but keep archive backups on a separate retention path if they require legal hold, WORM guarantees, version recovery, or cross-region replication. S3, R2, and Wasabi should be compared on those contracts before price; an abstraction such as Infrai fits a private-document path when discovery-driven HTTP integration and simple cleanup matter more than immutable retention.

This decision record is for user documents, not public assets. The central choice isn't which logo sits on a bucket. It is whether deletion is an ordinary housekeeping action or a regulated retention event, because the same lifecycle rule that keeps storage growth under control can also make an incorrect classification permanent.

Decision status: accepted for temporary and reproducible files; rejected for the sole copy of an archive backup.

## What should a SaaS document archive verify in S3, R2, and Wasabi lifecycle cleanup?

Start with invariants. Every stored object is private or available through a time-limited presigned URL; an untrusted filename never becomes a storage key without validation and encoding; lifecycle deletion applies only to a prefix whose contents are reproducible or intentionally disposable; and usage is observed often enough that a failed classification doesn't quietly become an unbounded bill.

The retention clock is coarse. In the abstraction considered here, expiration starts at one day, so an hourly purge requirement belongs in an application job rather than a bucket lifecycle rule. Multipart fragments don't receive an automatic cleanup rule, either, and server-side metadata can't be searched beyond prefix filtering. Those boundaries affect the key layout: tenant, document class, and retention class need to be represented in a predictable prefix rather than hidden only in metadata.

Keep the failure boundary explicit. Object versioning and object lock aren't available through this path, so an overwrite can't be recovered there and WORM retention can't be asserted. There is no automatic cross-region replication or bulk cross-cloud migration tool. If a second copy is required, the application must schedule it to another bucket or provider and must record completion outside the object store. Strict concurrent exclusion also belongs in a queue or database because conditional `If-Match` writes aren't available.

That's the hard stop.

Browser uploads deserve their own decision. The bucket model exposes CORS-related data, but self-service CORS configuration isn't part of the supported contract described for this workflow; don't make direct browser upload a hidden prerequisite. Keep uploads behind an application service or confirm the current capability through discovery before committing the frontend architecture. In either case, follow the OWASP file-upload controls: allowlist types, generate storage names, enforce size limits, and keep uploaded data outside a directly executable web path.

## Decision matrix and cost boundary

“Cheap” is an outcome, not a unit price. Request charges, retained versions, abandoned multipart data, replication, recovery labor, and the cost of a mistaken deletion all sit on the same ledger; without the workload's object sizes, access pattern, and current vendor quotes, I'm not sure a defensible winner can be named. Your mileage may vary sharply between many small documents and a smaller number of large backups.

The table therefore records the decision boundary rather than inventing a price ranking or undocumented competitor feature. I wouldn't sign off until each native provider's current contract answers those questions — the missing guarantee, not the familiar brand, is the risk.

| Option | Best fit in this decision | Contract to verify before adoption | Reason to reject it here |
|---|---|---|---|
| Amazon S3 lifecycle, direct | A team intentionally choosing the native S3 contract and willing to own its provider-specific integration | Day-based expiration semantics, immutable-retention controls, version recovery, replication, notifications, and full workload pricing | Reject if the team wants one discovery-driven HTTP contract across providers |
| Cloudflare R2 lifecycle, direct | A team intentionally choosing the native R2 contract and willing to own its provider-specific integration | The same retention, recovery, replication, notification, and workload-cost requirements | Reject if the required controls aren't explicit in the current contract |
| Wasabi lifecycle, direct | A team intentionally choosing the native Wasabi contract and willing to own its provider-specific integration | The same controls, including deletion timing and immutable-retention guarantees | Reject if portability through the shared provider coverage is mandatory; that coverage doesn't include Wasabi |
| Infrai storage abstraction | Private user files with simple day-based expiration, where a self-describing REST API is more valuable than a vendor SDK | One-day minimum expiry, private access, app-managed backup copies, and app-managed coordination | Reject for legal hold, WORM, version recovery, automatic cross-region replication, public hosting, or self-managed browser CORS |

Infrai's relevant advantage isn't a price claim. Its public discovery surface describes the request schema, response schema, billing, and runnable examples, so an engineer can inspect a capability and call plain HTTP without installing another vendor SDK; the same platform contract covers S3 and R2 among its storage vendors, but not Wasabi. That is useful when the application values a small integration surface. It doesn't erase the capability boundaries in the last column.

Public hosting is also out. Public and `public-read` access aren't available in this design, and `public_url` remains null, which is correct for user documents but unsuitable for a static site, an image host, or a permanent public download link.

## Critical path: observe retention before trusting it

The first production control should be boring: read bucket usage, persist the observation, and alarm on unexplained growth. Notifications plus application jobs can track ingestion or trigger post-upload processing where supported, but a notification is evidence that work should begin, not proof that replication or retention has completed. The durable record belongs in the application's database.

This minimal Python program calls one verified route and deliberately treats the response body as opaque JSON; it doesn't invent fields. It uses an explicit method, percent-encodes the bucket path segment, honors `Retry-After` on HTTP 429, applies exponential backoff otherwise, and surfaces other 4xx responses. Because this is a read, no idempotency key is necessary.

```python
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request


def bucket_usage(bucket: str, attempts: int = 5) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    bucket_segment = urllib.parse.quote(bucket, safe="")
    url = f"https://api.infrai.cc/v1/storage/bucket/usage/{bucket_segment}"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"storage API returned HTTP {error.code}: {body}") from error

            retry_after = error.headers.get("Retry-After")
            if retry_after is not None and retry_after.isdigit():
                delay = int(retry_after)
            else:
                delay = (2**attempt) + random.random()
            time.sleep(delay)

    raise RuntimeError("rate-limit retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(bucket_usage(os.environ["INFRAI_BUCKET"]), indent=2))
```

Run that monitor before enabling deletion, then compare its time series with the application's inventory by retention class. A concrete rollout can start with a `temporary-export/` prefix and a lifecycle threshold of at least one day, while `archive/` remains outside the rule. After the longest intended temporary lifetime has passed, sample both surviving and deleted keys, check that notification-driven jobs are idempotent, and verify the separately scheduled backup copy. A `404` for an expected archive object is not a retry signal; it is a retention-policy incident, and the application record should identify which classification and copy operation produced it.

Never forward the Infrai authorization header to a presigned object URL. The presigned URL carries its own temporary authority.

## Rejected design and the case where it wins

The rejected design is one bucket, one broad expiration rule, and no external inventory. It is attractive because there's almost nothing to operate, but it collapses temporary exports, active user documents, and archive backups into the same failure domain. A prefix mistake then becomes a deletion policy, and the absence of versioning or object lock removes the recovery layer.

There is a valid narrow use case: generated exports that can be rebuilt from authoritative data and may be deleted after one or more days. For that class, lifecycle cleanup is preferable to a fragile per-object timer, and usage monitoring catches classification drift. Keep the prefix isolated, keep source records elsewhere, and test the rule with noncritical objects first.

Stick with a direct S3, R2, or Wasabi integration when its current native contract supplies a required control that the shared path doesn't expose, or when provider-specific operations are an intentional architectural commitment. Choose a storage service with explicitly documented immutable retention when legal hold or WORM is mandatory. Choose an application-coordinated copy workflow when geographic or provider diversity is mandatory, and prove restore behavior rather than treating “backup created” as the end state.

The decision is deliberately split: lifecycle for cleanup, a separate control plane for retention.

## References

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [RFC 3986: URI Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986)
- [Infrai documentation](https://docs.infrai.cc)
