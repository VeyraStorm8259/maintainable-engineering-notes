# School Moderation Text Summarization API — 6 Records Behind Chat Completions

TL;DR: A Node.js text summarization API built on chat completions should never make its generated synopsis the record of truth. Keep six linked records—source, segment, attempt, output, review, and allocation—then let compact JSON output accelerate human triage of a long article or report. This costs more storage and forces explicit retention rules, but it preserves the evidence chain while making model activity attributable to each tenant.

A report may combine a lesson transcript, student comments, quoted messages, and the reporter's interpretation. Compression can help a reviewer find the allegation; classification can place the report in a queue. Neither operation should silently replace the submitted material or decide enforcement. The constraint is therefore provenance first, with per-tenant cost visibility derived from the same append-only execution history rather than reconstructed from a monthly invoice.

## How should a Node.js text summarization API store chat completions?

The source report survives, byte for byte, under an immutable version identifier. So do the segment offsets that entered each request. A summary without those coordinates is convenient prose, not durable evidence: a reviewer cannot distinguish an omission in the model output from text that never reached the model.

Use six records with separate lifecycles:

| Record | Minimum useful fields | Why it remains separate |
|---|---|---|
| Source | tenant, report ID, version, content hash, retention class | Establishes what was submitted |
| Segment | source version, ordinal, start and end offsets, segment hash | Makes splitting reproducible |
| Attempt | logical operation ID, attempt number, runtime, model, timestamps, outcome | Exposes retries and uncertain executions |
| Output | attempt ID, schema version, validated JSON, input references | Prevents an invalid response becoming evidence |
| Review | output generation, reviewer disposition, correction reason | Separates human judgment from automation |
| Allocation | tenant, attempt ID, reported usage, estimated usage, catalog version | Supports reconciliation without rewriting history |

That is more metadata than a single `summary` column. It is deliberate. Source retention may be shorter or more restricted than operational metrics; generated text may need deletion when its source expires; allocation records can retain counts without retaining sensitive prose. Separate records let those policies be expressed without pretending one database row has one natural lifetime. The trade-off is operational weight: six linked objects demand referential-integrity checks, deletion orchestration, and a way to detect an orphaned output. For a small internal tool that summarizes public articles, has one tenant, and carries no moderation consequence, this design is probably inappropriate; a validated response plus short-lived request metadata may be enough. The full evidence chain earns its keep when reviewers act on sensitive reports and costs must be assigned across schools.

Short reports can take a single model call. Long reports need deterministic segmentation, usually on paragraph boundaries with a secondary split for an oversized paragraph. Character counts are only conservative admission controls because tokenization depends on the selected model. Output also consumes context. Store offsets and the splitting-policy version, and never infer them later from a regenerated chunk.

No overlap is a sensible default for paragraph-aligned material because overlap repeats allegations and consumes processing twice. Add overlap only when an evaluation set shows boundary omissions, then account for the duplicate input and teach the reducer not to count repeated evidence twice.

Measure before adding it.

## Make JSON a validation boundary

JSON syntax is the beginning of the contract, not the contract itself. An HTTP success can still contain an unknown category, an extra key, a summary that exceeds the reviewer interface, or fluent text unsupported by the report. Validate locally before persisting an accepted output.

The following Python example deliberately uses a generic chat transport even though the surrounding service may be Node.js. It shows the storage-facing contract for an API that must summarize text, not a vendor SDK recipe. Endpoint, model identifier, credentials, and supported structured-output controls remain deployment configuration; those details cannot be assumed to be portable across runtimes.

```python
from __future__ import annotations

import hashlib
import json
from dataclasses import dataclass
from typing import Any
from urllib.request import Request, urlopen

ALLOWED_CATEGORIES = {"harassment", "self_harm", "privacy", "other"}


@dataclass(frozen=True)
class AcceptedOutput:
    operation_id: str
    summary: str
    categories: tuple[str, ...]
    input_tokens: int | None
    output_tokens: int | None


def operation_id(tenant_id: str, source_version: str, segment: int) -> str:
    material = f"{tenant_id}:{source_version}:{segment}:triage-v4".encode()
    return hashlib.sha256(material).hexdigest()


def validate_result(value: Any) -> tuple[str, tuple[str, ...]]:
    if not isinstance(value, dict) or set(value) != {"summary", "categories"}:
        raise ValueError("result must contain exactly summary and categories")

    summary = value["summary"]
    categories = value["categories"]
    if not isinstance(summary, str) or not 1 <= len(summary) <= 900:
        raise ValueError("summary must contain 1..900 characters")
    if not isinstance(categories, list) or not all(
        isinstance(category, str) for category in categories
    ):
        raise ValueError("categories must be a list of strings")
    if not set(categories).issubset(ALLOWED_CATEGORIES):
        raise ValueError("result contains an unknown category")
    return summary, tuple(categories)


def summarize_segment(
    endpoint: str,
    api_key: str,
    model: str,
    tenant_id: str,
    source_version: str,
    segment_index: int,
    text: str,
) -> AcceptedOutput:
    op_id = operation_id(tenant_id, source_version, segment_index)
    payload = {
        "model": model,
        "response_format": {"type": "json_object"},
        "messages": [
            {
                "role": "system",
                "content": (
                    "Return JSON with exactly summary and categories. "
                    "Describe allegations neutrally and do not decide enforcement. "
                    "Categories are harassment, self_harm, privacy, or other."
                ),
            },
            {"role": "user", "content": text},
        ],
    }
    request = Request(
        endpoint,
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "X-Operation-Id": op_id,
        },
        method="POST",
    )
    with urlopen(request, timeout=45) as response:
        envelope = json.load(response)

    summary, categories = validate_result(
        json.loads(envelope["choices"][0]["message"]["content"])
    )
    usage = envelope.get("usage", {})
    return AcceptedOutput(
        operation_id=op_id,
        summary=summary,
        categories=categories,
        input_tokens=usage.get("prompt_tokens"),
        output_tokens=usage.get("completion_tokens"),
    )
```

The 900-character constraint is an application contract in this example, not a model limit. It must be tested against the reviewer interface and the amount of evidence reviewers actually need. Likewise, `json_object` describes one possible transport capability; schema conformance still comes from local validation, and a runtime's current documentation must be checked before using any structured-output field.

Keep report text out of ordinary logs. Identifiers, hashes, offsets, byte counts, timings, and outcome codes are generally enough to operate the pipeline, while a tracing backend is not automatically an approved store for student-related content.

## Failure modes are data states, not exceptions

A timeout creates an unknown outcome: the remote runtime may have completed work after the client stopped waiting. Record the attempt as uncertain. Retrying under the same logical operation ID connects both attempts, but it does not prove the first execution was free or cancelled. An idempotency header has meaning only where the receiver documents it.

Unknown isn't free.

Missing usage is unknown, not zero.

Preserve any provider-reported usage on every attempt, maintain a separately labeled estimate when reporting is absent, and reconcile the two later. Currency conversion belongs in the allocation layer with a versioned catalog; observed token counts should not be mutated because a rate schedule changes. For a self-hosted runtime, publish the allocation basis—such as active inference time, reserved capacity, or weighted processed units—because each assigns idle and contention costs differently.

Partial completion needs the same discipline. If 11 of 12 segment outputs validate, retain those 11, retry only the missing ordinal, and refuse reduction until the complete expected set exists. Letting a reducer accept a partial set produces plausible summaries with invisible holes. Repeated schema-invalid output should move to a reviewable terminal state after a bounded retry policy; sending the identical request indefinitely increases cost without changing the constraint.

Semantic failures are harder. Negation can disappear, quoted speech can be assigned to the reporter, and uncertainty can become an assertion even when the JSON is perfect. Evaluate allegation preservation, speaker attribution, negation, unsupported claims, abstention, and category recall. Slice results by category and tenant language patterns. A single aggregate score can conceal the rare safety case that matters most.

## Compare execution modes after fixing the records

Transport choice comes after the evidence model because synchronous calls, asynchronous batches, and self-hosted queues can all produce the same six records. Their operational boundaries differ.

| Execution mode | Appropriate constraint | Tenant-allocation consequence | Named failure mode |
|---|---|---|---|
| Synchronous hosted request | A reviewer is waiting and input is bounded | Usage can arrive with the response; timeouts require reconciliation | Completion after client timeout |
| Asynchronous batch | Reports can wait and arrivals are bursty | Submission, completion, and reported usage arrive at different times | Partial completion against a stale source generation |
| Self-hosted queue | Runtime and data placement are owned internally | Capacity and idle time require a declared allocation policy | Queue saturation or silent model drift |

Concrete products illustrate boundaries, not winners. OpenAI documents file-based asynchronous Batch processing. Amazon Bedrock documents model invocation logging controls, while Google Vertex AI documents batch prediction workflows. These interfaces and retention implications differ, so treat their documentation as evidence for particular mechanisms rather than as a common contract. The portable layer is the source generation, logical operation ID, accepted-output schema, and append-only allocation record.

Prompt guidance also cannot settle storage semantics. It can improve the chance of a useful response, but prompts don't supply immutability, retention enforcement, reconciliation, or referential integrity.

## Roll out the evidence chain in 4 steps

First, create source versions, segment manifests, and append-only attempt records while the existing human queue remains authoritative. Shadow generation should write results and allocation data without changing routing.

Second, test the JSON validator and semantic evaluation set against quoted speech, negation, long paragraphs, unknown categories, and incomplete segment sets. Fix thresholds before looking at rollout results. Otherwise the threshold tends to follow whatever the current model happened to produce.

Third, expose summaries to a small traffic slice as advisory material. Monitor schema rejection, missing usage, retry amplification, queue delay, corrections by taxonomy slice, and source-to-summary evidence access. Rollback is plain: stop new generation and show the immutable source.

Finally, permit generated categories to route reports only when reviewers can reach the cited source version and the tenant ledger reconciles attempts rather than successful responses alone. **The durable result is an auditable chain from submitted evidence to human disposition.** JSON is merely one record in that chain.

## Sources

- https://platform.openai.com/docs/guides/batch
- https://www.promptingguide.ai
- https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html
- https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/batch-prediction-from-cloud-storage
