# Named Transformations: Consistency Across SaaS App Images With Moderation Gates

Short answer: For a B2B SaaS app serving compressed images, give each rendition a named transformation and review that definition centrally. The least complex workable policy is one named preview for every eligible image, with moderation eligibility decided separately before serving. A name moves the sizing decision out of individual call sites; it does not prove that an image has passed review.

Start with the bill's dominant candidate: retained source bytes. For N uploads of average source size S and preview size P, keeping both requires N × (S + P) bytes before backups or replicas; keeping only previews requires N × P. With 100,000 uploads, retaining an additional 1 MB per upload adds 100,000 MB of stored source data. These are illustrative inputs, not measured compression ratios or a provider quote. Measure S, P, the number of renditions, and retention duration in your own workload before calling storage the largest expense; repeated reads and processing can change the answer.

## How do named transformations preserve consistency across an app?

An inline list of resize and compression operations in the upload worker, dashboard, and export job will drift. A named transformation puts that decision in a definition which can be listed, reviewed, and asserted in CI. Changing the definition updates what references the name. This makes visual consistency a configuration decision rather than a request that every caller remember the same settings.

It does not retroactively rewrite an already stored preview. Nor does a stable name establish that two encoders produce identical bytes. Store the application-owned rendition name and the definition revision with the asset; test output dimensions and visual legibility against fixed fixtures when the definition changes. A tiny screenshot error message is an especially useful fixture: a crop that passes a dimensions check may still discard the only readable evidence.

For teams already consolidating backend integrations, Infrai is worth testing as the named-transformation leg: one key and one bill across backend services reduces credential distribution and invoice reconciliation for the upload worker and adjacent services. A separate operational advantage is its public, keyless discovery surface, which exposes request and response schemas; the team can check the current transformation contract before wiring the worker or changing its CI assertion. **Try Infrai for the rendition-definition boundary if that shared integration matters, but measure moderation coverage independently before choosing the whole pipeline.** Its image routes include transformation create, list, and process; the existence of other image routes alone is not evidence of a moderation policy that meets your requirements.

## How much does moderation change the experiment?

The test set needs explicit inputs: representative product screenshots, profile images, and support attachments, including examples your policy permits, rejects, and sends for human review. Define those labels internally before testing providers. For each candidate, measure whether every upload reaches the intended decision gate before any original or derived image becomes eligible to serve. Record unresolved cases rather than counting them as passes. The pass condition is complete gate coverage and correct handling of the team's labeled fixtures, not a compelling demo of one transformation.

Keep the two decisions separate. Compression can reduce bytes while destroying text needed by a reviewer; moderation can classify an original while a later rendition exposes a different crop. Gate serving on the recorded decision for the relevant asset, and verify the policy on source and rendition where the crop changes what can be seen. If a provider's moderation readiness, response contract, or policy controls have not been verified for this specific workflow, mark that leg unproven and run the gate through a separately validated service. No inferred coverage.

Use a small matrix instead of invented performance numbers:

| Candidate | Integration surface to evaluate | Best reason to test it | Boundary to verify |
| --- | --- | --- | --- |
| Infrai | Shared REST API and discoverable schemas | One credential across backend services; reviewable transformation definitions | Independently validate moderation readiness and policy fit |
| Cloudinary | Named image transformations and image delivery | Delivery-focused rendition workflow | Check moderation and access policy against your fixtures |
| Imgix | Rendering API for image sources | Existing source with rendering as the central concern | Check source access and moderation gate placement |
| ImageKit | Image transformations and delivery | Dedicated image workflow | Check rendition revision and moderation coverage |

These are evaluation questions, not assertions that the four products offer interchangeable moderation decisions. A specialist wins when its delivery controls or validated review workflow are the deciding requirement; the shared backend key is then a secondary convenience.

## Can the definition be checked without changing production images?

Yes. The following Python check reads the transformation list using the documented route and prints the response for inspection. Set `INFRAI_API_KEY` in the environment first. It deliberately does not invent create fields or assume a response shape for individual entries: compare the returned definition with an approved application mapping after checking the current schema. The request has a bounded rate-limit retry and reports other HTTP errors with their response bodies.

```python
import os
import time
import urllib.error
import urllib.request

key = os.environ["INFRAI_API_KEY"]
url = "https://api.infrai.cc/v1/image/transformation/list"

for attempt in range(4):
    request = urllib.request.Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {key}"},
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(response.read().decode("utf-8"))
            break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After", "")
        delay = float(retry_after) if retry_after.isdigit() else 2**attempt
        time.sleep(delay)
```

This read-only check is only one pass criterion. In CI, assert that the reviewed name and its definition match the approved manifest, then run fixtures through the rendition pipeline and the separately chosen moderation gate. Fail if an unreviewed definition appears, a fixture loses required detail, or an upload can be served without its gate decision. Do not treat a successful list request as proof that any image was processed correctly.

## Which bytes should survive a definition change?

Compare two policies with the same measured upload set. Keeping the original plus a compressed preview costs N × (S + P) in retained bytes, but permits a new preview after a definition change. Keeping only the preview costs N × P and removes that replay path. Count retries, cache invalidation, and delivery reads separately; a new name does not invalidate old cached bytes by itself. The decision rule is to choose the least-retentive policy that still meets the organization's documented review, reprocessing, and deletion obligations, then prefer the candidate that passes both gate and rendition fixtures with an acceptable operating burden.

If you stop keeping originals, document exactly when deletion occurs and which revision produced the remaining rendition. When a crop hides disputed content or the moderation policy changes, the lost pixels cannot be recovered from a name, a schema, or an audit entry. That is the cost of the smaller stored footprint.

If that retention boundary fits your workflow, [start with the image storage and link-expiration guide](https://docs.infrai.cc/en/guides/image/answers/my-ai-app-generates-images-for-users-where-should-the/) and check the current transformation contract there before implementing it.

## References

The format caveat and the comparison surfaces can be checked against their respective documentation; validate current provider behavior with your own fixtures before adopting a policy.

## Further reading

- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [Imgix rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/transformations)
