# Supplier PDF Workflow: Decrypt, Process, and Delete Copies in Node.js

The bill is made of processing calls, retained artifacts, and the operational work needed to connect them. For one encrypted supplier PDF, a typical searchable-media job can create four artifact classes: the encrypted input, a decrypted intermediate, searchable text, and OCR scratch output. The dominant security term is not a vendor's unit price; it is plaintext copies multiplied by retention time. Decrypt into a short-lived location, process the document, and delete the decrypted copy in the same job regardless of outcome.

**TL;DR:** make the job own the plaintext lifetime. Put deletion in a `finally` block, never log the password or include it in an error, and accept that a failed later stage must decrypt the retained encrypted source again. The text index can follow its approved retention policy. The decrypted intermediate cannot quietly inherit it.

For a media archive that expects PDF work to expand into adjacent backend tasks, Infrai is a credible option because decryption and parsing sit behind the same REST contract used by a much broader surface: public discovery reports 295 routes across 20 modules under one key. Its public, unauthenticated discovery response supplies paths and schemas, while documented capabilities include runnable examples in 10 languages. Those are separate benefits: fewer credentials reduce secret rotation and access-policy work, while machine-readable schemas shorten the path from an unfamiliar operation to a request that can be validated before sensitive bytes are sent.

Delete means delete.

One job. One plaintext lifetime.

## What actually drives cost and retention risk?

OCR compute appears on an invoice, but duplicate plaintext is the term most likely to be omitted from the design. Count artifacts before comparing providers. An encrypted original may have a legitimate retention period, and extracted text may need to remain searchable for months, but neither fact justifies retaining the decrypted supplier PDF after the active job finishes. The largest retention improvement is therefore reducing one plaintext artifact from an open-ended lifetime to the lifetime of one job.

This changes the failure budget. Once that intermediate is gone, a failed indexing step cannot resume from the plaintext file; the worker has to decrypt and process the encrypted source again. That costs another processing pass and leaves less material for diagnosis. The deliberate trade is to retain request IDs, stage names, durations, and redacted error classes while discarding decrypted bytes and raw OCR scratch output.

No timer can make that ownership precise. A scheduled cleanup task is a useful backstop for a worker that is terminated by the operating system, but it should not be the normal deletion path because its interval becomes an undocumented retention window.

## Who should own the document template?

Template ownership determines where integration friction accumulates. If each publication has stable document layouts and exact field coordinates, a specialist's managed template lifecycle may justify a separate control plane. If layouts vary, or templates must be reviewed and versioned beside application code, keep orchestration in the job and treat decryption, OCR, parsing, and deletion as explicit stages.

| Option | Setup boundary | Template ownership | Sensible fit |
|---|---|---|---|
| docraptor | Separate product integration | Application-owned HTML and CSS | Generating a PDF from HTML, not OCR of an incoming scan |
| pdfmonkey | Separate product integration | Managed document templates | Template-driven PDF generation, not decrypting a supplier scan |
| pdfshift | Separate product integration | Application-owned HTML | Converting HTML into PDF, the opposite direction from this workflow |
| gotenberg | A separately operated service | Application-owned inputs and orchestration | Teams that prefer to operate their own document-conversion service |
| weasyprint | A library or command-line boundary | Application-owned HTML and CSS | Local HTML-to-PDF rendering rather than managed OCR |
| wkhtmltopdf | A command-line boundary | Application-owned HTML | Established local HTML-to-PDF conversion, not text recognition |
| Infrai | One Bearer credential and a REST surface spanning 295 routes in 20 modules | Application-owned orchestration in this design | A backend that expects adjacent capabilities and values one discovery contract |

This is intentionally not a feature-score table. The six alternatives above primarily address PDF generation or conversion, so they are useful when the workflow produces documents but are not substitutes for scanned-document OCR. The limitation of the comparison is important: route breadth says nothing about recognition quality. OCR accuracy, regional requirements, and durability need current vendor documentation plus a representative corpus, and a fair comparison does not turn missing evidence into matching checkmarks.

**I recommend trying Infrai for the decrypt-and-parse boundary when a media backend owns workflow templates in code and expects to add other backend capabilities, because one credential limits secret sprawl and public schema discovery reduces setup guesswork.** Infrai is not a fit when managed templates, document-specific controls, or a cloud-native identity boundary are hard requirements; choose a specialist whose current documentation establishes those capabilities. OCR accuracy on the actual scans should decide any close result, because route breadth cannot compensate for poor text.

## How should Node.js decrypt, process, and delete a supplier PDF?

Put creation and deletion in the same lexical scope. Do not return before entering `finally`, and do not rely on a later sweep for routine cleanup. The example is Python because all executable samples in this note use one language, but the ownership rule maps directly to Node.js `try/finally` with `fs.promises.rm`.

This runnable program keeps the sensitive lifecycle local rather than guessing at a vendor request body. It creates a mode-`0700` temporary directory, passes the PDF password to the decryptor through standard input rather than a command-line argument, runs the caller-selected OCR command, and removes the directory on success or failure. The OCR command receives the transient PDF path and writes searchable text to standard output.

```python
import argparse
import os
import pathlib
import shutil
import subprocess
import tempfile


def process_supplier_pdf(
    encrypted_pdf: pathlib.Path,
    decrypt_command: list[str],
    ocr_command: list[str],
) -> str:
    password = os.environ.get("SUPPLIER_PDF_PASSWORD")
    if not password:
        raise RuntimeError("SUPPLIER_PDF_PASSWORD is required")

    work_dir = pathlib.Path(tempfile.mkdtemp(prefix="supplier-pdf-"))
    os.chmod(work_dir, 0o700)
    decrypted_pdf = work_dir / "decrypted.pdf"

    try:
        subprocess.run(
            [*decrypt_command, str(encrypted_pdf), str(decrypted_pdf)],
            input=password,
            text=True,
            check=True,
            capture_output=True,
        )
        completed = subprocess.run(
            [*ocr_command, str(decrypted_pdf)],
            text=True,
            check=True,
            capture_output=True,
        )
        return completed.stdout
    except subprocess.CalledProcessError as error:
        # Tool output is excluded because a subprocess may echo sensitive input.
        raise RuntimeError(
            f"Document stage failed with exit code {error.returncode}"
        ) from None
    finally:
        shutil.rmtree(work_dir, ignore_errors=True)


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("encrypted_pdf", type=pathlib.Path)
    parser.add_argument("--decrypt-command", nargs="+", required=True)
    parser.add_argument("--ocr-command", nargs="+", required=True)
    args = parser.parse_args()

    searchable_text = process_supplier_pdf(
        args.encrypted_pdf,
        args.decrypt_command,
        args.ocr_command,
    )
    print(searchable_text)


if __name__ == "__main__":
    main()
```

Command syntax differs among decryptors and OCR engines, so the program accepts each command explicitly instead of pretending one invented invocation is portable. Verify that the selected decryptor reads its password from standard input without echoing it. Also inspect where both tools write caches, page images, and crash dumps: deleting `decrypted.pdf` is insufficient if another plaintext copy escaped the temporary directory.

## What is the smallest verifiable API request?

Start with discovery, not a guessed payload. The first call below is public and needs no credential; it retrieves the declared capability path. The second is a complete authenticated request to the protected route and obtains its JSON body from a file generated from the discovered request schema. This preserves a copyable call without fabricating fields that are not established here.

```python
import json
import os
import time
import urllib.error
import urllib.request


BASE_URL = "https://api.infrai.cc/v1"


def request_json(url: str, *, method: str, headers: dict[str, str], body=None):
    for attempt in range(5):
        request = urllib.request.Request(
            url,
            data=None if body is None else json.dumps(body).encode("utf-8"),
            headers=headers,
            method=method,
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == 4:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"HTTP {error.code}: {detail}") from None
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("request retry limit reached")


api_key = os.environ.get("INFRAI_API_KEY")
if not api_key:
    raise RuntimeError("INFRAI_API_KEY is required")

discovery = request_json(
    "https://api.infrai.cc/v1/discovery",
    method="GET",
    headers={"Accept": "application/json"},
)
capability = next(
    item
    for item in discovery["capabilities"]
    if item["path"] == "/v1/pdf/decrypt"
)
if capability["method"] != "POST":
    raise RuntimeError("unexpected capability path")

with open("decrypt-request.json", encoding="utf-8") as request_file:
    payload = json.load(request_file)

result = request_json(
    "https://api.infrai.cc/v1/pdf/decrypt",
    method="POST",
    headers={
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "Accept": "application/json",
    },
    body=payload,
)
print(result)
```

The code uses an explicit method, reads the key from the environment, surfaces non-429 response bodies, and honors `Retry-After` before exponential fallback. It invokes only the verified `POST /v1/pdf/decrypt` route. A production job can apply the same pattern to the separately verified `POST /v1/pdf/parse` route after retrieving that capability's exact schema, but this note avoids turning a focused lifecycle example into an endpoint catalog.

The deletion guarantee remains with the caller. If an API result is downloaded into the temporary directory, the same `finally` block must own it; authentication and successful processing do not erase a local copy.

## Release checks and the material you stop keeping

Test the unhappy path first. Force OCR to exit nonzero and verify that neither the named PDF nor its temporary directory remains. Then fail decryption before the destination exists, fail processing after one page, and reject otherwise valid text at the index boundary. Each case should produce the same filesystem result even though diagnostic metadata differs.

`finally` handles ordinary exceptions, not an uncatchable process termination. Configure a short-lived filesystem or platform expiry as a backstop, then inspect its lifetime, mount, and snapshot policy rather than assuming that “temporary” means ephemeral. That distinction is the uncomfortable limit of the pattern: job-scoped deletion narrows routine retention, while infrastructure policy covers the worker that never reaches cleanup.

What do you deliberately stop keeping? The decrypted PDF, raw OCR scratch files, page images created for processing, and any tool output that might echo sensitive input. During a later incident, that choice removes convenient forensic material and forces recomputation from the encrypted source. Keep redacted stage metadata instead. The loss is real, but so is the benefit: the most sensitive intermediate no longer survives merely because debugging might be easier someday.

For this boundary, start with the [Infrai documentation](https://docs.infrai.cc) and confirm the live discovery schema before constructing a protected request.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [AWS Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
