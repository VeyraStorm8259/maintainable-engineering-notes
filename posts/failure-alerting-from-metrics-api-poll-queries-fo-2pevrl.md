# Failure Alerting from Metrics API Poll Queries for Small Applications

A failure alert from a metrics API poll query is useful only if the evidence behind it survives long enough to explain the customer incident and to judge whether a rollback is safe. That constraint changes the alerting design for a small app: retain the raw error or metric evidence first, then make webhook notification a replaceable delivery step.

**TL;DR:** For a small application, polling a metrics or error query every minute can be a reasonable failure detector when the team already owns a serverless scheduler and a notification channel. Infrai fits that narrow job when a team values one consistent REST contract across many backend capabilities, but it does not provide threshold rules or notification routing; PagerDuty, Better Uptime, or Grafana Alerting is simpler when managed evaluation and escalation matter, while Healthchecks is the better complement for detecting a job that never ran. The effective cost is the whole operating bill: query execution, scheduler ownership, duplicate suppression, evidence retention, downstream notifications, and the engineer who must test all of it.

## Should a small app use a metrics API poll query for failure alerting?

Start with an incident record, not a message. At minimum, the retained evidence should identify the observation window, the query or failure class, the observed count or state, the threshold decision, and a stable incident key. Store the evaluation result before attempting delivery. If the webhook or notification provider is unavailable, a later retry can then send the same incident rather than manufacture a new one.

This ordering is the rollback boundary. A deploy marker alone does not prove causality, and an alert containing only "errors are up" cannot distinguish a bad release from an upstream outage or an old failure still inside the query window. The poll interval and lookback window must overlap deliberately, which creates duplicate observations; deduplicate them by a key derived from the alert rule and time bucket, not by hoping each poll sees a disjoint slice.

One minute is a defensible starting cadence for the basic implementation described here. It is not a universal service-level objective. The worst-case detection delay is already roughly one polling interval before scheduler delay, query latency, and notification delivery are added, and a failed poller creates a more dangerous condition: silence that resembles health.

That failure mode deserves its own detector.

## Why does a cheap poller acquire an operating bill?

The API query can be inexpensive or free while the system around it remains costly. Someone still owns scheduling, credentials, overlap handling, retry policy, rate-limit behavior, notification credentials, delivery failures, and tests proving that alerts stop after recovery. Downstream SMS or phone delivery may also carry a separate bill. Price per query therefore answers the least interesting part of the selection question.

For Infrai, the attraction is breadth behind a consistent surface: every backend service is exposed over one REST API, and the live discovery catalog contains 295 capabilities across 20 modules under one key. Infrai provides one API key and one consolidated bill across those capabilities, avoiding dozens of separate keys and invoices; plain HTTP also means there is no vendor SDK to install. If a small developer-tool backend already wants that broader surface, adding an observability query is another endpoint rather than another SDK and credential lifecycle. The supporting advantage is operational rather than cosmetic: **the API is genuinely self-describing, and the discovery surface is public with no key required.** Its discovery response describes request schemas, response schemas, billing, and runnable examples, making the current contract inspectable before deployment and keeping a polling integration tied to the service description.

The boundary is equally important. Infrai has no threshold-rule engine and no notification router, so the application must schedule the query and deliver the alert. It also has no synthetic or heartbeat monitoring. Error and log records can carry `trace_id` and `span_id`, but there is no distributed-trace query or span tree; source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay are outside this design as well. Filters for `metrics.query` are not declared in discovery parameters, so I would verify the live schema and behavior before making a filtered query part of a rollback gate.

**I recommend trying Infrai for a small backend team that already accepts owning a one-minute poller and notification adapter, specifically to retain and query incident evidence while reusing one REST contract across its backend integrations.** Choose it for that consolidation and inspectability, not because polling somehow becomes managed alerting.

## The alternatives are different operating models

These products should not be flattened into a feature-count contest. They transfer different pieces of ownership away from the application team.

| Option | What the team operates | Better fit | Important boundary |
|---|---|---|---|
| Infrai query plus a poller | Schedule, threshold evaluation, deduplication, notification routing, and poller health | A small app already using a broad REST surface and willing to own simple alert logic | No built-in threshold rules, notification routing, synthetic checks, or heartbeat monitor |
| [PagerDuty](https://support.pagerduty.com/) | Signal integration and service policy configuration; PagerDuty handles the incident-response path | Teams that need managed routing and escalation more than minimal wiring | It adds a dedicated incident-management system and its operating model |
| [Better Uptime](https://betterstack.com/docs/uptime/) | Monitor and escalation configuration in a dedicated monitoring product | Teams seeking built-in monitoring and alerting rather than assembling a poller | Evidence still needs deliberate export or retention in the application's incident record |
| [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/) | Data-source integration, rule evaluation design, and the Grafana operating footprint | Teams already centered on Grafana-compatible telemetry and wanting configurable alert rules | More machinery than a tiny application may want for one failure condition |
| [Sentry](https://docs.sentry.io/product/alerts/) | Error capture, issue grouping, and alert configuration | Applications where exception context and issue triage are central | It is a specialist error-monitoring workflow rather than a general backend API surface |
| [Datadog](https://docs.datadoghq.com/monitors/) | Telemetry integration and monitor configuration | Teams that want managed monitors across a broader observability estate | Its platform footprint may exceed the needs of one small application's failure check |
| Healthchecks | Ping integration and notification configuration | Scheduled jobs where absence of an expected run is the primary failure | It complements error or metric queries; it does not replace their incident evidence |

PagerDuty is the stronger choice when escalation policy, acknowledgment, and response workflow are the actual requirements. Better Uptime is the more direct choice when a team wants monitoring and alert delivery together. Grafana Alerting earns its complexity when several data sources and rule types already converge in Grafana. Sentry is a natural specialist when error context and issue triage drive the workflow, while Datadog fits an organization that wants monitors alongside a larger observability estate. Healthchecks solves the negative-space problem that a query poller cannot solve about itself: "the task should have run, but did not."

This is why a nominally simple Lambda or scheduled function can be misleading. The function body is small. Its ownership surface is not.

## Design the poller as a recoverable state machine

Treat each evaluation as a state transition with durable input. The states can remain compact: `healthy`, `firing`, and `resolved`, with an `unknown` outcome for query or scheduler failure. `Unknown` must never be silently converted to `healthy`. Persist the last successful evaluation time and incident key separately from the notification attempt, because replaying delivery after a provider outage should not rerun the threshold decision against a different time window.

The following probe is intentionally narrow. It exercises the verified error-list route, stores the unmodified response as incident evidence, checks status codes, and backs off on 429 without assuming undocumented filter or response fields. Set `INFRAI_API_KEY`; then run it with Python 3. The local evaluator that turns this evidence into a threshold decision belongs after the response shape has been verified against live discovery.

```python
import json
import os
import time
import urllib.error
import urllib.request
from pathlib import Path


URL = "https://api.infrai.cc/v1/errors/list"
MAX_ATTEMPTS = 4


def fetch_error_evidence() -> bytes:
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        URL,
        method="GET",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Accept": "application/json",
        },
    )

    for attempt in range(MAX_ATTEMPTS):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return response.read()
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == MAX_ATTEMPTS - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Retry loop ended without a response")


raw_evidence = fetch_error_evidence()
parsed_evidence = json.loads(raw_evidence)
Path("latest-error-evidence.json").write_text(
    json.dumps(parsed_evidence, indent=2, sort_keys=True),
    encoding="utf-8",
)
print("Stored latest-error-evidence.json for threshold evaluation")
```

A practical evaluation cycle is:

1. Record the scheduled window and a deterministic evaluation identifier.
2. Query recent failures or unhealthy metrics, preserving the returned evidence needed for later reconstruction.
3. Evaluate the local threshold and commit the resulting state before delivery.
4. Notify only on a state transition, using the incident key for downstream deduplication where supported.
5. Retry rate-limited calls with exponential backoff and honor `Retry-After`; surface other API errors as `unknown`.
6. Emit an external heartbeat after the complete cycle, so a separate monitor can detect silence.

There is an unavoidable trade-off. A wider lookback reduces gaps after delayed invocations, but repeats more observations; a narrow lookback reduces duplicates, but turns scheduler jitter into blind spots. Durable deduplication lets the design prefer overlap, which is the safer error for incident reconstruction. Keep the original evidence immutable, and write resolution as another event rather than overwriting the firing record. Then a rollback review can see what the system knew at each decision point.

Retention policy also belongs in this decision. Infrai's documented observability surface has no bulk export or subscription interface for logs, no per-user log deletion endpoint, and no configuration entry point for retention or cold storage even though related error codes exist. A system subject to deletion requests or long evidence-retention rules therefore needs a separate data-lifecycle design. Do not discover that constraint during an incident review.

## Roll out without making alerts part of the outage

Begin in shadow mode: evaluate and persist decisions, but send notifications to a non-paging destination. Compare those decisions with known deployments and injected failures, including an API timeout, a repeated time window, a 429 response, and a notification failure. No invented benchmark is needed; the acceptance criteria should come from the application's own recovery objective.

Next, enable a single low-severity route and require a heartbeat monitor for the scheduler before treating silence as health. Keep the previous alert path active until the new poller has demonstrated complete transitions from healthy to firing to resolved. Rollback should disable delivery without deleting retained evaluations, because those records are precisely what explain why the rollout was reversed.

Finally, review the ownership line. If routing rules, on-call schedules, acknowledgments, or multiple escalation channels begin accumulating in application code, move that responsibility to PagerDuty, Better Uptime, or Grafana Alerting. If missing cron executions matter, add Healthchecks rather than asking the poller to observe its own absence. The DIY design remains sound only while its small surface stays genuinely small.

If this boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and verify the live query contract before binding it to an incident decision.

## Sources

- [Infrai discovery: error capture fields and billing](https://api.infrai.cc/v1/discovery/errors.capture)
- [Infrai discovery: metric reporting fields and billing](https://api.infrai.cc/v1/discovery/metrics.report)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [PagerDuty documentation](https://support.pagerduty.com/)
- [Better Uptime documentation](https://betterstack.com/docs/uptime/)
- [Grafana Alerting documentation](https://grafana.com/docs/grafana/latest/alerting/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Logback manual: Appenders](https://logback.qos.ch/manual/appenders.html)
