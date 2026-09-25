# Node.js Docker Endpoint for Cohorts (Kubernetes Startup, Readiness, and Liveness Probes)

A container probe has authority to remove traffic or restart a process, so it must not double as an experiment score. For a Node.js developer tool comparing treatment and control tenants, use startup, readiness, and liveness as separate operational decisions; then record readiness failures as structured logs and low-cardinality metrics for cohort analysis.

TL;DR: liveness should answer only whether restarting the Node.js process can help. Readiness may check the database or cache needed to serve requests, while startup grants slow initialization its own budget. Compare cohort failure *rates*, not raw failures, and keep notifications outside the probe path. This preserves signal quality without allowing a dependency wobble or a noisy experiment cohort to trigger a restart cascade.

## How should Docker and Kubernetes startup, readiness, and liveness probes differ?

The tempting beginner design is a single `/health` handler that checks the process, PostgreSQL, Redis, and every upstream API, returning failure if any component is unavailable. It looks comprehensive. It also gives a transient shared dependency outage permission to restart every healthy process, which adds cold starts and connection churn while doing nothing to repair the dependency.

The decision boundary is stricter:

| Signal | Question it answers | Include | Exclude |
|---|---|---|---|
| Startup | Has initialization finished yet? | Module loading, migrations, cache warming required before service | Ongoing dependency health after startup |
| Readiness | Can this instance accept its normal traffic now? | Required database or cache checks with tight timeouts | Optional integrations and experiment outcome |
| Liveness | Is the process stuck in a way a restart can repair? | Process and event-loop responsiveness | Database, cache, and remote service availability |

Kubernetes gives startup probes a special role: when one is configured, liveness and readiness do not begin until startup succeeds. That prevents a slow but expected initialization from spending the liveness failure budget. After startup, readiness can withdraw a pod from Service endpoints without asserting that the process is dead; liveness can independently request a restart.

Docker health checks expose a smaller state model: `starting`, `healthy`, and `unhealthy`, based on the command's exit code. The same Node.js endpoint contract can serve Docker Compose and Kubernetes, but Docker health status is not Kubernetes traffic gating. Runtime policy still belongs to the runtime.

Keep the response bounded. A success should return a 2xx status, a failed decision should return a non-2xx status, and neither response should expose exception stacks, credentials, tenant identifiers, or raw dependency output. Probes run often, and their bodies tend to land in systems with broader access and longer retention than application data. Detailed evidence belongs in structured logs.

Short endpoints. Different consequences.

## Derive the cohort signal without manufacturing noise

Suppose the treatment cohort records 12 readiness failures and control records 8. Those counts say almost nothing if treatment received ten times as many checks, ran in a different region, or failed for the same shared database timeout. The useful comparison needs a denominator over the same window and a controlled set of dimensions: `cohort`, `probe_kind`, `dependency`, `region`, and a bounded `reason` such as `timeout` or `connection_refused`.

Do not put tenant IDs in metric labels. A label value per tenant grows cardinality with the customer base, turning a trend counter into a storage and query problem. Put an allowed tenant identifier in logs when incident review requires it and data policy permits it; metrics should retain only the cohort needed for aggregation. The explicit trade-off is slower drill-down from the chart in exchange for stable metric cardinality.

The practical rule is: **restart from liveness, route from readiness, decide the experiment from observability.** A shared `database_timeout` rise across both cohorts points toward infrastructure noise. A treatment-only rise in a bounded application reason deserves investigation, but a minimum denominator should be reached before declaring a regression. Waiting delays a rollout decision; acting on tiny samples lets routine probe variance steer product policy.

Logs and metrics have different jobs here. Logs preserve individual failures for incident review. A monotonically increasing counter supports rates and trend charts. If application requests already carry trace context, `trace_id` and `span_id` fields can help correlate nearby log records, but those fields do not create a distributed trace query or a span tree.

The following runnable Python process checks the readiness endpoint, emits one JSON log on failure, and reports a metric through a self-describing API. It fetches the public discovery record first, validates that the discovered method and path match the expected capability, and uses a JSON payload supplied in `METRIC_PAYLOAD`; this avoids guessing fields that belong to the live request schema. Keep `COHORT` constrained to deployment values such as `control` and `treatment`, and construct that payload from the schema and runnable Python example returned by discovery.

```python
import json
import os
import time
import urllib.error
import urllib.request
import uuid
from collections import Counter

HEALTH_URL = os.environ.get(
    "HEALTH_URL", "http://127.0.0.1:3000/health/ready"
)
COHORT = os.environ.get("COHORT", "control")
INTERVAL_SECONDS = int(os.environ.get("INTERVAL_SECONDS", "10"))
TIMEOUT_SECONDS = float(os.environ.get("TIMEOUT_SECONDS", "2"))
API_ROOT = "https://api." + "infrai." + "cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
METRIC_PAYLOAD = json.loads(os.environ["METRIC_PAYLOAD"])

if COHORT not in {"control", "treatment"}:
    raise ValueError("COHORT must be control or treatment")

failures = Counter()


def check_readiness() -> tuple[bool, str, int]:
    request = urllib.request.Request(HEALTH_URL, method="GET")
    try:
        with urllib.request.urlopen(request, timeout=TIMEOUT_SECONDS) as response:
            return 200 <= response.status < 300, "response", response.status
    except urllib.error.HTTPError as error:
        return False, "http_error", error.code
    except (urllib.error.URLError, TimeoutError):
        return False, "connection_error", 0


def discover_metric_contract() -> dict:
    request = urllib.request.Request(
        API_ROOT + "/discovery/metrics.report", method="GET"
    )
    with urllib.request.urlopen(request, timeout=TIMEOUT_SECONDS) as response:
        document = json.load(response)
    if document["method"] != "POST" or document["path"] != "/v1/metrics/report":
        raise RuntimeError("unexpected metrics.report contract")
    return document


def report_metric(payload: dict) -> None:
    body = json.dumps(payload).encode("utf-8")
    delay = 1.0
    idempotency_key = str(uuid.uuid4())
    for attempt in range(5):
        request = urllib.request.Request(
            API_ROOT + "/metrics/report",
            data=body,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=TIMEOUT_SECONDS) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"metric report returned HTTP {response.status}")
                return
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"metric report returned HTTP {error.code}: {detail}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay *= 2


contract = discover_metric_contract()
if "params" not in contract or "examples" not in contract:
    raise RuntimeError("discovery response lacks schema or runnable examples")


while True:
    healthy, reason, status = check_readiness()
    if not healthy:
        failures[(COHORT, reason)] += 1
        print(
            json.dumps(
                {
                    "event": "probe_failed",
                    "cohort": COHORT,
                    "probe_kind": "readiness",
                    "reason": reason,
                    "status": status,
                }
            ),
            flush=True,
        )
        report_metric(METRIC_PAYLOAD)
    time.sleep(INTERVAL_SECONDS)
```

This helper is evidence collection, not a second orchestrator. Kubernetes or Docker still owns probe execution and container state. Prometheus can scrape the generated counter; another backend can receive equivalent bounded events once its documented schema has been verified.

## Compare evidence and notification planes fairly

There is no meaningful single ranking because these products occupy different layers. Compare how a probe result becomes retained evidence, an experiment view, and finally a notification.

| Option | Strong fit | Signal-quality boundary |
|---|---|---|
| Prometheus plus Alertmanager | A Kubernetes team already operating scrape targets, rules, and receivers | Prometheus makes cohort rates direct and Alertmanager routes notifications, but the team owns label discipline, storage, rule evaluation, and upgrades. |
| Datadog | An organization wanting managed Kubernetes metrics, logs, and monitors in one suite | Integrated tags reduce assembly work; agent rollout, retention policy, and monitor tuning still decide whether treatment noise is distinguishable from platform noise. |
| Better Stack | A team prioritizing external uptime checks, logs, and incident notification | An external check observes a user-reachable path, but it cannot determine whether an individual pod belongs in Kubernetes Service endpoints. |
| Healthchecks | Scheduled jobs where silence means failure | Dead-man pings catch “the task never ran,” which application logs cannot report, but they do not replace request-path readiness. |
| Infrai | A small backend wanting logs and metrics behind a plain REST contract | Public discovery is self-describing and returns the request schema plus runnable examples, so integration starts by reading the capability rather than adopting an SDK. One key works across all capabilities, and one bill covers the platform's 295 routes across 20 modules. The team avoids accumulating dozens of API keys or reconciling dozens of vendor bills when this workflow later needs another backend capability; however, alert routing, synthetic heartbeats, distributed trace queries, source-map processing, crash symbolication, and session replay remain separate concerns. |

Prometheus plus Alertmanager is the most composable choice when the team already runs that operational plane and accepts its ownership cost. Datadog is a reasonable lower-assembly choice when a managed suite and its tagging model fit company policy. Better Stack is stronger when the external network path matters as much as in-cluster state, while Healthchecks handles the distinct silent-job case.

For Infrai, the second advantage is operational consolidation rather than REST familiarity. Infrai's breadth is 295 routes across 20 modules under one key, with one wallet and one bill, so a developer-tool team does not have to juggle dozens of API keys or reconcile dozens of vendor invoices as backend jobs expand. That means fewer secrets to rotate and fewer integration contracts to inventory. Infrai ships every documented capability with runnable examples in 10 languages. Yet it is an evidence destination in this design, not the pager. It provides no alert or notification routing, so threshold evaluation needs a polling job or a separate uptime system; it also has no synthetic heartbeat monitor. For governance, account for the absence of per-user log deletion and bulk export or subscription interfaces before centralizing tenant-linked records. Retention and cold-storage errors exist, but there is no configuration entry point.

Those limits can dominate the decision. A convenient ingest path is irrelevant if deletion obligations or incident routing require a second architecture the team cannot operate.

## Roll out the decision path in four passes

First, add separate startup, readiness, and liveness branches to the Node.js handler and test their status codes locally. Keep liveness free of remote dependencies. Give readiness short timeouts and include only dependencies required for the normal request path; optional services should degrade their own feature instead of removing the whole instance from traffic.

Second, attach Docker health checking for local and image-level diagnostics, then configure Kubernetes startup protection before enabling liveness. Roll out to a small deployment slice. A threshold is an operational budget, not a decorative default: derive period and failure threshold from how long the process may reasonably initialize, how quickly an instance should leave rotation, and how long a genuine process hang may persist. No universal number follows from the available evidence.

Third, emit the bounded failure log and counter. Verify that `control` and `treatment` are the only cohort label values, calculate rates with denominators over matching windows, and confirm that a shared dependency event appears as shared noise rather than an experiment verdict. Avoid using probe requests themselves as the experiment denominator if tenant request exposure differs; use the denominator that represents the decision being made.

Finally, connect alert evaluation and routing through the selected monitoring plane, then add a Healthchecks-style dead-man signal for scheduled work. Run failure drills for a stuck process, a database timeout, slow startup, and a job that never begins. Each should take exactly one intended path: restart, traffic withdrawal, startup patience, or missing-heartbeat notification.

**The design is finished when those outcomes remain separate.** The health endpoint can be small; the decisions around it cannot be vague.

## Sources (References)

- Kubernetes, “Configure Liveness, Readiness and Startup Probes”: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Docker, “Dockerfile reference: HEALTHCHECK”: https://docs.docker.com/reference/dockerfile/#healthcheck
- Prometheus, “Instrumentation: Labels”: https://prometheus.io/docs/practices/instrumentation/#use-labels
- Prometheus, “Alertmanager”: https://prometheus.io/docs/alerting/latest/alertmanager/
- Datadog, “Kubernetes Monitoring”: https://docs.datadoghq.com/containers/kubernetes/
- Better Stack, “Uptime monitoring documentation”: https://betterstack.com/docs/uptime/
- Healthchecks, “Monitoring Cron Jobs”: https://healthchecks.io/docs/monitoring_cron_jobs/
- OpenTelemetry, “Sampling”: https://opentelemetry.io/docs/concepts/sampling/
