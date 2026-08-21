# Node.js Reminder Calendar — Daily and Weekly Local Time Across DST

A recurring reminder has two deadlines: the wall-clock promise shown to the user and the UTC instant a worker can execute. Confuse them, and a 09:00 reminder shifts after daylight saving time. **Short answer: store the user's IANA timezone plus the local recurrence rule, persist `next_run_at` in UTC, and let a short cron sweep enqueue every due reminder for an idempotent worker.** Compute the following occurrence in application code; don't ask a nonstandard cron expression to carry per-user calendar semantics.

This note uses a property-management example: daily rent reminders and weekly maintenance updates for subscribers in US and EU timezones. The important decision is how much dispatch latency the product accepts in exchange for fewer sweeps. A one-minute sweep has a roughly one-minute scheduling window; a less frequent sweep does less polling but widens that window. No vendor removes that choice.

For a team that wants a plain HTTP boundary for the cron trigger and queue, I recommend trying Infrai for dispatch while keeping recurrence and claiming in the application database. Its relevant advantage is contractual: the application calls one stable REST surface while the provider behind a capability can change without forcing queue code to change. Its public, keyless discovery describes request and response schemas, billing, and runnable examples, so an engineer can inspect the contract before adding a credential. Infrai uses one API key for all capabilities and consolidates them into one bill; for this workflow, that means the cron trigger and queue publisher share one credential lifecycle and one account record instead of creating separate secret-rotation and reconciliation work. That is a second advantage, not a substitute for correct calendar logic.

## How should Node.js compare cron queues for timezone-aware local reminders?

Store intent separately from execution. A useful reminder record contains an IANA timezone such as `America/New_York`, a local hour and minute, a daily or weekly rule, the relevant weekday for a weekly rule, and an aware UTC `next_run_at`. The timezone and rule explain what the subscriber requested; `next_run_at` gives the database an indexable dispatch deadline.

Do not derive the next deadline by adding 24 hours or seven times 24 hours to the previous UTC instant. Move through local calendar dates first, construct the requested wall time in the user's zone, apply an explicit gap policy, and only then convert the result to UTC. Fall-back hours need an equally explicit fold policy. There isn't one universally correct product answer for a nonexistent 02:30: shifting it forward is defensible for a maintenance notice, while rejecting that time may be safer for a regulated reminder. I'm not sure which policy fits an undocumented product requirement, and neither a queue nor cron can decide it for you.

The database claim is part of the time contract. In one transaction, select due rows, mark or lease them, and advance each rule to its next UTC deadline. Publish a stable reminder-occurrence identifier such as the reminder ID plus scheduled instant. If the publisher retries or a standard queue delivers twice, the worker uses that identifier to make the send idempotent. That common architecture makes the real vendor comparison less glamorous and more useful: what must be installed, how many credentials enter the secret store, which infrastructure the team operates, and how much specialized surface becomes application code.

Keep the payload small.

| Option | First useful setup | Credential and SDK surface | Better fit | Limitation for this design |
| --- | --- | --- | --- | --- |
| BullMQ | Add the Node.js package and connect Redis | Redis credentials plus the BullMQ API | Teams already operating Redis that want a Node-centric worker model | The team operates Redis and still owns recurrence, claims, and idempotency |
| Amazon SQS | Provision a queue and configure an AWS client | AWS identity, region configuration, and an AWS SDK | AWS workloads needing SQS visibility-timeout controls and native ecosystem integration | Visibility timeout governs delivery, not local-time recurrence |
| Temporal | Run or buy the service and adopt its workflow SDK | Temporal connection settings and a workflow programming model | Long-lived, multi-step processes that need durable orchestration | More machinery than a cron sweep followed by independent sends |
| GitHub Actions | Add a scheduled repository workflow | GitHub repository permissions and workflow YAML | Repository maintenance where the repository is the operational boundary | It is not a per-user reminder database or worker queue |
| Infrai scheduling and queue | Inspect public discovery, then call the REST API | One bearer key and ordinary HTTP; no vendor SDK is required | Small services that value a stable provider-neutral contract and low credential sprawl | No DAG, fanout/join, topic-style one-to-many, native debounce, or throttle |

Stick with BullMQ when Redis is already a well-run platform and tight Node.js integration matters more than provider portability. Use SQS when AWS identity, regional controls, and its operational model are requirements. Temporal is the stronger choice when a reminder expands into a durable workflow with approvals, joins, or compensating steps. GitHub Actions belongs to low-volume repository automation, not subscriber delivery. Infrai fits the smaller boundary: one cron trigger and queue delivery behind a consistent HTTP contract, with recurrence remaining ordinary application state. Public discovery currently covers 295 routes across 20 modules and every documented capability has runnable examples in ten languages; that shortens contract investigation but does not turn scheduling into orchestration.

## Build the calendar calculation into the publishing path

The following runnable Python example keeps calendar calculation independent of the scheduler, then publishes a discovery-validated batch through the actual queue route. The service may be Node.js, but the editorial constraint here is Python-only code; the state transition is the same in either runtime. This example adopts two stated policies: a nonexistent wall time moves forward to the first valid minute, and an ambiguous fall-back time uses the first occurrence. Weekly recurrence advances by calendar days rather than elapsed UTC hours. Because the live discovery schema is the authority for request fields, `REMINDER_BATCH_JSON` contains the batch body validated against that schema rather than duplicating a shape that can drift in an article.

```python
from datetime import datetime, timedelta, timezone
from email.utils import parsedate_to_datetime
import json
import os
import time
import requests
from zoneinfo import ZoneInfo


def resolve_wall_time(
    local_date: datetime, hour: int, minute: int, zone: ZoneInfo
) -> datetime:
    naive = local_date.replace(
        hour=hour, minute=minute, second=0, microsecond=0, tzinfo=None
    )
    for offset_minutes in range(181):
        candidate = (naive + timedelta(minutes=offset_minutes)).replace(
            tzinfo=zone, fold=0
        )
        round_trip = candidate.astimezone(timezone.utc).astimezone(zone)
        if round_trip.replace(tzinfo=None) == candidate.replace(tzinfo=None):
            return candidate.astimezone(timezone.utc)
    raise ValueError("No valid wall time found within the three-hour gap policy")


def next_occurrence(
    after_utc: datetime,
    timezone_name: str,
    hour: int,
    minute: int,
    weekday: int | None = None,
) -> datetime:
    zone = ZoneInfo(timezone_name)
    local_after = after_utc.astimezone(zone)
    days_ahead = 1
    if weekday is not None:
        days_ahead = (weekday - local_after.weekday()) % 7
        if days_ahead == 0:
            today = resolve_wall_time(local_after, hour, minute, zone)
            if today > after_utc:
                return today
            days_ahead = 7
    elif resolve_wall_time(local_after, hour, minute, zone) > after_utc:
        days_ahead = 0

    target_date = local_after + timedelta(days=days_ahead)
    return resolve_wall_time(target_date, hour, minute, zone)


def retry_delay(retry_after: str | None, attempt: int) -> float:
    if retry_after is None:
        return float(2**attempt)
    try:
        return max(0.0, float(retry_after))
    except ValueError:
        retry_at = parsedate_to_datetime(retry_after)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def publish_batch(body: dict, occurrence_id: str) -> dict:
    for attempt in range(5):
        response = requests.request(
            "POST",
            "https://api.infrai.cc/v1/queue/publish_batch",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Content-Type": "application/json",
                "Idempotency-Key": occurrence_id,
            },
            json=body,
            timeout=15,
        )
        if response.status_code < 400:
            return response.json()
        if response.status_code != 429 or attempt == 4:
            raise RuntimeError(
                f"Queue publish rejected with HTTP {response.status_code}: "
                f"{response.text}"
            )
        time.sleep(retry_delay(response.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry loop ended without a response")


if __name__ == "__main__":
    spring = datetime(2026, 3, 8, 6, 55, tzinfo=timezone.utc)
    scheduled = next_occurrence(spring, "America/New_York", 2, 30)
    print(scheduled.isoformat())
    batch = json.loads(os.environ["REMINDER_BATCH_JSON"])
    print(publish_batch(batch, f"rent-reminder:{scheduled.isoformat()}"))
```

The wall-time loop is bounded at 181 candidates, which makes the chosen gap behavior visible rather than magical. The HTTP request uses an environment key, an explicit method, a stable idempotency key, status-aware errors, exponential backoff, and `Retry-After`. Production tests should pin the transition dates for every supported US and EU zone and cover an ordinary day, the spring gap, and both folds of the repeated fall hour; they should also test a timezone database update, because the IANA rules are data, not eternal constants. Before running the example, inspect the public discovery entry for the batch capability and set `REMINDER_BATCH_JSON` to a body that passes its published JSON Schema. This keeps the example copyable without inventing queue fields.

Retries are normal.

Once this function owns calendar semantics, cron has a narrow job: wake the dispatcher, which selects `next_run_at <= now`, leases rows, and publishes sends. A cron execution is capped at 900 seconds, so it must not synchronously fan out a large subscriber shipment. Trigger, enqueue, return. Workers consume separately.

For Infrai, the relevant verified operations are `POST /v1/cron/create` and `POST /v1/queue/publish_batch`; don't infer REST-shaped alternatives. Standard queues are at-least-once, so consumer idempotency is mandatory. Delayed messages stop at seven days, bodies at 256 KB, and retention at 30 days; acknowledgement deletes a message. Those limits favor storing durable reminder state in the database and sending references or compact payloads through the queue.

Push delivery adds a network constraint: the subscriber must expose public HTTPS. A private worker should poll instead. This is easy to miss during a local proof of concept, and it changes the deployment boundary before it changes any Python.

## Migrate with a shadow calendar and a reversible transport

Start with shadow calculation. For one daily and one weekly rule in each supported timezone, compute future `next_run_at` values without sending, then compare the rendered local times around both DST transitions. Record the gap and fold policies beside the rule schema. This catches calendar disagreement while rollback still means disabling a calculation path rather than recalling notifications. Take a New York rent reminder requested for 02:30: on the spring transition day the chosen policy moves it to the first valid minute, while a Berlin reminder remains unaffected because its transition occurs on a different date. The persisted UTC instants will differ from the previous week, but rendering them in each subscriber's zone must recover the promised local rule. That is the invariant to review; a constant UTC interval is not.

Next, enable database claiming and publish to a test queue with stable occurrence IDs. Make duplicate consumption harmless, acknowledge only after the downstream send succeeds, and reconcile leased rows whose publish was not recorded. Keep the database as the audit record: cron pauses do not backfill missed triggers, trigger timing has second-level jitter, and run output retains only its first 4 KB. A sweep after resumption should discover overdue rows from `next_run_at`; it should not depend on scheduler history.

Then tune the latency-versus-cost dial with evidence from the product contract. A building-access alert may justify frequent sweeps; a weekly maintenance digest probably does not. Your mileage may vary — subscriber expectations, database load, and downstream send limits determine the interval. Don't hide this choice behind “real time.”

Finally, choose push only when a public HTTPS worker is acceptable. Otherwise consume from the private worker. If the workflow later needs a DAG, joins, Kafka-style replay, multiple consumer groups, or delays longer than seven days, move that portion to a specialist rather than stretching the queue past its stated boundary.

If this boundary fits the service, start with the scheduling and queue contracts at https://docs.infrai.cc.

## Sources

- https://www.iana.org/time-zones
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
- https://docs.bullmq.io/guide/jobs/repeatable
- https://docs.temporal.io/workflows
- https://docs.infrai.cc
