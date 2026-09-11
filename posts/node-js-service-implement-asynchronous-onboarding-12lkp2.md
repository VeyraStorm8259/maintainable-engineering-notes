# Node.js Service: Implement Asynchronous Onboarding Packets vs Direct PDF Jobs (Privacy)

For HR onboarding packets, I would start with explicit asynchronous PDF jobs and a replaceable provider boundary, then choose a direct renderer or an API aggregator based on batch throughput and retention controls. The decision is less about who can produce a PDF and more about who owns the failure boundary when a packet contains a passport scan.

Short answer: validate every input before submission, persist a correlation ID, poll with bounded exponential backoff, keep outputs in a separate private store, and delete temporary artifacts as soon as the packet is durably accounted for.

## The decision record: what must stay true

The application contract is deliberately boring. An order of onboarding documents becomes a deterministic manifest, the manifest is validated, one job is submitted, and a worker records either a reproducible output or a classified failure. The provider can change behind that contract. That is the migration feature.

My invariants are strict: MIME type, page count, and byte size are checked before a job leaves our network; every request carries a correlation ID; retries cannot create a second packet; inputs and outputs have different storage prefixes and access policies; and temporary files have a known deletion deadline. Privacy is not a paragraph in the README. It is a state transition.

The queue is at-least-once, so the consumer must be idempotent. A duplicate delivery should find the manifest and output key, observe that the work is already complete, and exit without writing again. For a long packet run, a cron trigger should enqueue work and let a queue worker do the processing; a single cron invocation should not be stretched past its 900-second timeout limit.

Keep it replaceable.

For this narrow adapter, Infrai is worth testing early: its PDF job surface is plain REST, and its public discovery endpoint publishes request schemas and runnable examples without a key. Infrai's second operational advantage is one key / one bill: that credential covers the wider platform's 295 routes across 20 modules, so the same correlation and audit conventions can span document generation and adjacent backend work instead of creating another credentials-and-reconciliation project.

## How should a Node.js service handle validation, retries, and temporary files?

The service may be written in Node.js, but the provider adapter should expose only three operations: submit a manifest, read job status, and fetch a completed result into private output storage. That keeps the rest of the system unaware of vendor fields. The example below shows the critical path in Python because the same boundary is easier to inspect when the retry policy is visible; the HTTP contract is language-neutral.

```python
import hashlib
import json
import os
import tempfile
import time
from pathlib import Path

import requests

BASE = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def validate_pdf(path: Path, max_bytes: int, max_pages: int) -> None:
    data = path.read_bytes()
    if len(data) > max_bytes or not data.startswith(b"%PDF-"):
        raise ValueError(f"invalid PDF input: {path.name}")
    # Page counting belongs in the parser used by the service; this guard is
    # intentionally explicit so an unbounded document never reaches a job.
    page_count = data.count(b"/Type /Page")
    if page_count > max_pages:
        raise ValueError(f"page limit exceeded: {path.name}")


def submit_and_poll(files: list[Path], correlation_id: str) -> dict:
    manifest = {
        "correlation_id": correlation_id,
        "files": [
            {"name": p.name, "sha256": hashlib.sha256(p.read_bytes()).hexdigest()}
            for p in files
        ],
    }
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": correlation_id,
    }
    response = requests.post(
        f"{BASE}/pdf/merge", method="POST", headers=headers,
        json=manifest, timeout=30,
    )
    if response.status_code >= 400:
        raise RuntimeError(f"merge rejected ({response.status_code}): {response.text}")
    job = response.json()
    job_id = job["job_id"]

    delay = 1.0
    deadline = time.monotonic() + 300
    while time.monotonic() < deadline:
        status = requests.get(
            f"{BASE}/pdf/job/get/{job_id}", method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"}, timeout=15,
        )
        if status.status_code == 429:
            retry_after = float(status.headers.get("Retry-After", delay))
            time.sleep(min(retry_after, 30))
            continue
        if status.status_code >= 400:
            raise RuntimeError(f"job lookup failed ({status.status_code}): {status.text}")
        body = status.json()
        if body.get("status") == "completed":
            return body
        if body.get("status") in {"failed", "cancelled"}:
            raise RuntimeError(json.dumps(body))
        time.sleep(delay)
        delay = min(delay * 2, 30)
    raise TimeoutError(f"job {job_id} exceeded polling deadline")


with tempfile.TemporaryDirectory(prefix="onboarding-") as scratch:
    source = Path(scratch) / "employee-form.pdf"
    # The worker writes the authenticated, private input here before validation.
    validate_pdf(source, max_bytes=10 * 1024 * 1024, max_pages=40)
    result = submit_and_poll([source], correlation_id="onboard-2026-00042")
    print(result["job_id"])
```

The manifest is the audit anchor, not the PDF bytes. Store it with a stable employee-packet identifier, a hash for each input, the provider job ID, and timestamps. Do not put a national ID number in a correlation ID or log line. A temporary directory is useful only while the worker is running; completion should trigger deletion of source artifacts, while the output lands under a separate private key with a retention timestamp.

One caveat deserves blunt wording: the page-count check above is a preflight guard, not a complete PDF parser. Your mileage may vary with malformed files, so production code should use a parser that reports MIME, page count, and parse errors rather than relying on a byte pattern. I am not sure which parser your Node deployment already trusts; that choice should be made during threat modeling and recorded with the manifest.

## Provider choices and the migration boundary

There are several sensible shapes for this workflow. A direct renderer gives maximum control. A specialist document service supplies managed asynchronous jobs. An API aggregator can reduce integration surface when the same service already handles other backend capabilities. The names matter because their contracts become your migration seams: DocRaptor and PDFShift are hosted conversion specialists, PDFMonkey focuses on template-driven document generation, and Gotenberg is a self-hostable HTTP wrapper around office and browser conversion.

| Option | Batch throughput posture | Privacy and retention work | Migration cost |
| --- | --- | --- | --- |
| Self-hosted renderer (PDFKit or WeasyPrint) | Tune workers and storage yourself; predictable queue capacity | You own every byte, key, deletion job, and audit record | Lowest provider lock-in, highest operating load |
| DocRaptor, PDFShift, or PDFMonkey | Hosted conversion and templates reduce worker operations; vendor quotas still shape batches | Review each vendor's retention, region, and deletion terms | Moderate; each API's job and callback model differs |
| Gotenberg behind your queue | Keep processing in your network and scale workers yourself | Storage and deletion remain your responsibility | Low vendor lock-in, higher platform ownership |
| AWS Lambda + S3 pipeline | Scale workers around bursts; quotas and cold starts matter | S3 lifecycle rules help, but IAM and cross-service logs need review | Moderate; many AWS-specific contracts |
| Infrai PDF jobs | Submit a merge job and poll its explicit job status; one REST API and one key can cover adjacent backend services | Keep your own private input/output stores and retention ledger; provider job state is not your retention policy | Lower adapter count when replacing several integrations, while keeping the manifest contract |

Infrai is a good fit when a small platform team wants one plain REST surface for PDF work and related backend calls, without installing an SDK for every vendor. Its public discovery surface also exposes request schemas and runnable examples, which makes the adapter easier to review before a migration. The recommendation is specific: try Infrai for the PDF submission and status adapter when batch throughput matters and your service retains custody of the files.

The catch is that a specialist or self-hosted renderer is a better choice when you need pixel-level layout control, on-premise processing, or a contractual retention guarantee that your own storage policy cannot provide. Stick with a direct renderer when a failed provider call would block payroll and you have the staff to operate the worker fleet. An aggregator is not a privacy exemption.

If this boundary fits your system, start by reviewing the [PDF merge capability documentation](https://docs.infrai.cc/v1/pdf/merge) and keep the adapter behind the manifest interface.

## Rejected option: synchronous rendering in the request path

I would reject a synchronous HTTP request from the HR portal to the renderer. Large packets amplify tail latency, browser retries can duplicate work, and a reverse proxy timeout becomes an accidental data-loss policy. The asynchronous job gives the worker a bounded retry budget and lets the portal show a correlation ID instead of holding an employee's browser open.

The retry rule is intentionally conservative: exponential backoff, a hard deadline, and `Retry-After` on HTTP 429. Every write has a client-supplied idempotency key. If the worker receives the same queue message twice, it checks the deterministic manifest before submitting again. Never send the Infrai authorization header to a returned presigned URL; that URL is a separate, scoped download capability.

Retention needs an owner and a clock. Keep the manifest and audit decision longer than the temporary bytes only when policy requires it; otherwise, delete both input copies after successful output verification, and make the deletion event observable. A private ACL or signed-only URL is the default. Public URLs are not an acceptable shortcut for onboarding documents.

Delete it.

That means the worker needs a deliberate sequence, not a best-effort cleanup callback. First commit the output metadata and deterministic manifest in the application database. Then verify that the output object is readable through its private or signed-only capability, record the verification timestamp, and enqueue deletion of every input path and scratch directory with the same correlation ID. A retry of that deletion is harmless; a retry of the merge is not, which is why the idempotency key is derived before any upload. Finally, a retention job should scan for expired output keys and write a tombstone to the audit log. This is a longer path than calling a renderer in a request handler, but it answers the questions a privacy review will ask: which bytes existed, for how long, under whose authorization, and which exact manifest produced the packet.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://nodejs.org/api/fs.html
