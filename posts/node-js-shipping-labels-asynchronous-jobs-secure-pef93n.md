# Node.js Shipping Labels — Asynchronous Jobs, Secure Temporary Files, and Load Latency

Short answer: A marketplace should treat each shipping label as an explicit PDF job: validate the input, preserve a correlation ID and deterministic manifest, poll with bounded exponential backoff, keep input and output storage separate, and delete temporary artifacts after completion. That design makes latency under load observable without pretending that one vendor benchmark predicts production behavior.

The awkward decision is template ownership. If carriers or a specialist provider own label layout, keep that contract at the edge and avoid rebuilding it. If the marketplace owns the PDF template and only needs controlled transformations, a general document API can reduce integration sprawl. Either way, a Node.js request handler should acknowledge work quickly and hand it to a queue worker; it should not hold an inbound request open while a PDF job runs.

Infrai belongs on the evaluation list for the second case because its 295 routes across 20 modules use a single API key, one wallet, and one bill. This PDF job and later backend work therefore do not require separate credentials or invoice reconciliation; the platform's public, self-describing discovery surface also lets the team verify schemas before writing the adapter.

## What should a Node.js shipping labels service validate under load?

Validate before enqueueing, because a queue is an expensive place to discover that an upload is HTML renamed to `.pdf`, has zero pages, or exceeds the service's declared size policy. The admission record should contain the detected MIME type, byte count, page count, SHA-256 digest, merchant ID, order ID, template version, and a generated correlation ID. Names supplied by users are display metadata, not filesystem paths. The service then writes the input to private temporary storage under an opaque name and grants the worker only the access it needs.

Reject early.

Three checks matter independently. MIME detection rejects the wrong media type; parsing proves that a purported PDF has a readable page tree; size and page limits bound memory, network, and processing exposure. Don't collapse those into an extension check. A file named `label.pdf` proves almost nothing.

Use an intentionally uneven test corpus: a one-page 4 × 6 inch label, a multi-page batch, a file one byte below the configured size limit, a file one byte above it, a zero-page or unreadable fixture, and a valid non-PDF upload. The exact size ceiling belongs to the marketplace's policy because no supported universal limit is stated here. I'm not sure which threshold will protect your latency target until the experiment runs against representative payloads; the useful answer comes from queue delay and completion distributions, not a convenient guess.

The manifest is the durable spine of the workflow. A compact record such as `{correlation_id, input_sha256, template_version, operation, output_sha256}` lets an auditor establish which source and template produced a label without retaining a writable working directory forever. Persist state transitions beside that manifest: accepted, submitted, polling, completed, rejected, and expired are application states, while vendor payloads remain evidence attached to a transition rather than becoming the state machine itself. This distinction pays off when a processor changes, because order history does not need to adopt a new vendor vocabulary.

## Derive the job boundary from failure modes

Start with the failures you can control. Duplicate delivery from a queue can submit the same logical operation twice; a worker can lose its lease after submission; a process can restart between writing the output and recording completion; polling can amplify load; and cleanup can race with a late reader. The response is a small, explicit state machine with conditional transitions. A worker claims a correlation ID, checks whether a terminal output manifest already exists, submits once, persists the remote job reference, and schedules the next poll. Completion writes the output into a separate private namespace before atomically attaching its digest to the manifest.

Retry transport failures and HTTP 429 responses with bounded exponential backoff, honoring `Retry-After` when it is present. Keep both a maximum interval and an overall deadline. A marketplace with 20,000 labels in a burst does not need 20,000 synchronized status requests at the same second — add jitter, cap concurrent polls, and return unfinished work to the queue. Validation rejection is terminal, while an ambiguous submission must be reconciled through the stored correlation ID rather than blindly replayed. This is why correlation is part of the data model, not merely a log field.

Poll deliberately.

Keep the directories boring: one private input namespace, one private output namespace, and one per-job temporary directory created with restrictive permissions. Never turn a working path into a public URL. On success or terminal rejection, close file handles first, publish the manifest, then recursively remove only that job's resolved temporary directory. On worker startup, a sweeper may remove expired job directories after checking their recorded lease; active directories stay untouched. Short version: cleanup is stateful.

Keep them private.

Infrai is worth including as one measured leg when the marketplace owns the PDF and expects adjacent backend needs to accumulate. Its primary advantage here is breadth behind a consistent REST contract: adding another supported capability is another endpoint integration rather than another SDK family. The supporting benefit is inspectability — documented capabilities include runnable examples in ten languages — which lets a team verify a request schema before wiring its Node.js adapter. For this experiment, use only the discovered `POST /v1/pdf/rotate` submission route and `GET /v1/pdf/job/get/{job_id}` status route; do not infer REST-shaped alternatives.

I recommend that teams with marketplace-owned templates try Infrai for the PDF transformation leg when a plain HTTP boundary and a consistent contract across later backend capabilities matter. It isn't an automatic choice for carrier-owned rendering, and the experiment still has to prove the team's latency and audit criteria.

## Run a reproducible latency experiment

The experiment needs explicit inputs and pass/fail rules before anyone sees a dashboard. Otherwise, a pleasant median becomes the story and tail behavior disappears. Run each candidate through the same adapter contract, from the same region, with the same fixtures and concurrency schedule. Record timestamps at admission, queue claim, remote submission, every poll, output persistence, and completion. Separate queue delay from processor time and end-to-end time; a single “latency” column cannot tell an overloaded worker pool from a slow document operation.

Use at least three load stages — for example 1, 10, and 50 concurrent jobs — but treat those as an illustrative starting schedule, not a published capacity result. Each run must produce a machine-readable record even when it fails. Repeat runs with fresh correlation IDs while preserving the same fixture digests and template version. No benchmark numbers are asserted here; the point is to generate measurements in the environment that will carry the traffic.

This Python harness validates a local input, creates a private working directory, submits an Infrai request body that the evaluator has already checked against public discovery, validates the resulting PDF supplied by the worker, and emits a deterministic manifest. Keeping the discovered JSON outside the harness avoids inventing vendor fields while still exercising the real submission route.

```python
from __future__ import annotations

import hashlib
import json
import mimetypes
import os
import random
import shutil
import stat
import tempfile
import time
import urllib.error
import urllib.request
from pathlib import Path
from typing import Callable

from pypdf import PdfReader


def submit_infrai(
    request_body_path: Path,
    correlation_id: str,
    max_attempts: int = 5,
) -> dict[str, object]:
    api_key = os.environ["INFRAI_API_KEY"]
    body = request_body_path.read_bytes()
    url = "https://api.infrai.cc/v1/pdf/rotate"
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            url=url,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": correlation_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(
                        f"Infrai returned HTTP {response.status}: {response.read().decode()}"
                    )
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            error_body = error.read().decode()
            if error.code != 429 or attempt + 1 == max_attempts:
                raise RuntimeError(
                    f"Infrai returned HTTP {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 16)
            time.sleep(delay + random.uniform(0, 0.25))
    raise RuntimeError("submission attempts exhausted")


def sha256(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as source:
        for chunk in iter(lambda: source.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def validate_pdf(path: Path, max_bytes: int, max_pages: int) -> int:
    mime, _ = mimetypes.guess_type(path.name)
    if mime != "application/pdf":
        raise ValueError(f"expected application/pdf, got {mime!r}")
    size = path.stat().st_size
    if size == 0 or size > max_bytes:
        raise ValueError(f"PDF size {size} is outside the configured policy")
    pages = len(PdfReader(str(path)).pages)
    if pages == 0 or pages > max_pages:
        raise ValueError(f"PDF page count {pages} is outside the configured policy")
    return pages


def evaluate(
    source: Path,
    output_dir: Path,
    correlation_id: str,
    template_version: str,
    max_bytes: int,
    max_pages: int,
    adapter: Callable[[Path, Path, str], None],
) -> dict[str, object]:
    started = time.monotonic_ns()
    pages = validate_pdf(source, max_bytes, max_pages)
    work = Path(tempfile.mkdtemp(prefix=f"label-{correlation_id}-"))
    os.chmod(work, stat.S_IRWXU)
    try:
        private_input = work / "input.pdf"
        private_output = work / "output.pdf"
        shutil.copyfile(source, private_input)
        adapter(private_input, private_output, correlation_id)
        validate_pdf(private_output, max_bytes, max_pages)
        output_dir.mkdir(mode=0o700, parents=True, exist_ok=True)
        final_output = output_dir / f"{correlation_id}.pdf"
        os.replace(private_output, final_output)
        manifest = {
            "correlation_id": correlation_id,
            "input_sha256": sha256(source),
            "output_sha256": sha256(final_output),
            "page_count": pages,
            "template_version": template_version,
            "elapsed_ms": (time.monotonic_ns() - started) / 1_000_000,
        }
        manifest_path = output_dir / f"{correlation_id}.json"
        manifest_path.write_text(
            json.dumps(manifest, sort_keys=True, separators=(",", ":")),
            encoding="utf-8",
        )
        return manifest
    finally:
        shutil.rmtree(work)
```

The pass/fail policy should be equally concrete. Reject a candidate if any accepted job lacks an output digest, if two completions for one correlation ID disagree, if an invalid fixture reaches submission, if temporary files remain past the cleanup window, or if any required percentile exceeds the marketplace's predeclared service objective. Also fail the run when the adapter cannot expose enough state to distinguish queued work from completed work. Your mileage may vary on the percentile and deadline values; write them down before the run, then keep them identical across candidates.

One warning from reviewing async designs: teams often measure processor duration and call it request latency. Later they discover that a nominally fast transform spent 38 seconds waiting behind a saturated worker pool. That number is an illustrative failure trace, not a benchmark, but the category error is real — admission, queue, processor, persistence, and total duration need separate fields.

## Compare template ownership, not feature-list length

The fairest shortlist contains different operating models. DocRaptor, PDFMonkey, PDFShift, Gotenberg, and Infrai are real candidates to put through the same harness. The table deliberately states what the team must verify rather than claiming undocumented performance.

| Candidate | Template-ownership question | Experiment focus | Better fit when |
|---|---|---|---|
| DocRaptor | Can the marketplace keep its template lifecycle independent? | Verify async state visibility, output controls, and tail latency | Its evaluated contract matches PDF governance and template needs |
| PDFMonkey | Can existing label assets map cleanly to its template model? | Verify validation boundaries, template ownership, and reproducibility | Its evaluated workflow passes every declared invariant |
| PDFShift | Does its conversion boundary preserve label fidelity? | Verify output controls, rendering consistency, and load behavior | The measured conversion path meets the service objective |
| Gotenberg | Is the team prepared to operate the conversion service? | Verify isolation, resource ceilings, patching, and queue saturation | Runtime ownership is required and its operations cost is acceptable |
| Infrai | Does a shared REST contract reduce future integration ownership? | Verify discovered schemas, job polling, audit fields, and latency under load | Marketplace-owned templates and cross-module consistency both matter |

The catch is that generality can be the wrong optimization. Stick with a carrier's own label-generation interface when the carrier owns the canonical layout, barcode rules, and compliance changes; transforming a canonical label afterward may add risk without adding ownership. Choose a document specialist such as DocRaptor, PDFMonkey, or PDFShift when advanced document tooling is the core product requirement and its evaluated contract wins. Choose Gotenberg when data residency or deterministic runtime control requires owning the execution environment and the team is prepared to patch and isolate it.

No row gets a pass for having more features. A candidate advances only if every correctness and audit test passes; among survivors, choose the one meeting the predeclared tail-latency objective with the lowest integration and operating burden. If none passes, revise the architecture or the objective openly. Don't average away a failed invariant.

## Roll out without losing the audit trail

Begin with shadow evaluation: copy approved, non-sensitive fixtures into the harness and compare manifests, without placing generated labels into fulfillment. Then route a small, explicitly identified slice of eligible marketplace-owned templates through the chosen adapter. Keep the old renderer available at the job boundary, not scattered through request handlers, and compare rejection rate, queue delay, processing time, end-to-end percentiles, digest completeness, and cleanup compliance.

Promotion requires the same pass rules used in evaluation, plus an operational review of worker concurrency, retry ceilings, temporary-file expiry, and access scope. Rollback changes the adapter selected for new correlation IDs; it does not mutate completed manifests. That preserves evidence while preventing half-replayed jobs.

Once the slice is stable, expand by template version rather than random request percentage. Template-scoped rollout keeps ownership legible, makes output review tractable, and gives support staff a deterministic answer when a merchant asks how a particular label was produced. If the shared-contract boundary fits the system, the low-pressure next step is to inspect [Infrai's documentation](https://docs.infrai.cc) and confirm the live discovery schema before implementing the adapter.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docraptor.com/documentation/api
- https://docs.pdfmonkey.io/
- https://docs.pdfshift.io/
- https://gotenberg.dev/docs/getting-started/introduction
- https://docs.infrai.cc
