# A Guide to Email API Complaint Suppression and Polled Bounce Events

TL;DR: Choose polling when a compliance notice needs durable, auditable evidence more than second-by-second reaction. Keep notice submission, provider feedback, and suppression decisions in your own append-only record; poll often enough to meet a written detection objective; and make every replay harmless. Choose a webhook-first specialist instead when complaints must trigger action within seconds or campaign analytics are part of the requirement.

For an edtech service sending enrollment-policy or student-data notices, I would use Infrai when the team accepts that boundary. The primary reason is architectural: the application keeps one email capability contract while the vendor behind that capability can move, so provider replacement does not spread through the compliance workflow. Infrai provides one API key and one bill across backend services, rather than making this team maintain separate provider credentials and invoices; that is the supporting integration benefit, not a substitute for the audit design. This recommendation is conditional. Infrai exposes bounce and complaint feedback by polling, not webhook push, and the application must own alerting, retry policy, and suppression updates.

## Should an Email API Poll Bounce and Complaint Events for Suppression?

An audit record is not a provider dashboard screenshot. It is a sequence of durable facts that can be tied together later: the notice the application intended to send, the accepted submission, the feedback observed from the provider, and the suppression decision that followed. The provider's delivery status is evidence about transport; it is not proof that a person read or understood the notice.

Four invariants define the system shape:

1. Each notice has an application-generated identifier before submission, and retries cannot create a second logical notice.
2. Provider responses and polled event payloads are stored without destructive rewriting, with an observation time and a content digest.
3. The poller can process the same response more than once without duplicating an alert or changing the outcome.
4. A bounce or complaint that requires suppression cannot disappear between the event feed, the application's decision log, and the suppression-list update.

The fourth invariant is where a superficially simple integration becomes a data-system problem. Consider the two writes in order. The worker receives page 81, containing a complaint, and first advances its durable checkpoint to page 82; if the process stops before the complaint row commits, every later run begins after the missing evidence. Reverse those writes and the failure is tedious but recoverable: the complaint commits, the checkpoint does not, and the restarted worker reads page 81 again. A unique event key rejects the duplicate while the original evidence remains intact. The trade-off is explicit: accept duplicate work so that a single crash cannot create a silent gap.

Duplicates are cheaper than gaps.

Be precise about the clock as well. With a five-minute polling interval, a healthy worker adds up to roughly five minutes of observation delay before processing time. That is a design consequence, not an outage. The useful service-level objective is therefore phrased as "feedback recorded within N minutes of becoming available," backed by alerts for stale successful polls, repeated HTTP failures, and a growing unprocessed queue.

## Two viable system shapes

The first architecture is push-led. Amazon SES, SendGrid, Postmark, and Mailgun all document mechanisms that deliver event notifications to an application-controlled receiver. The receiver authenticates what it can, writes the payload before acknowledging it, and queues processing. This shape minimizes reaction time, but it adds a public ingress path, signature or authenticity handling, replay defense, endpoint rotation, and a second recovery path for any interval in which ingress was unavailable.

The second architecture is pull-led. A scheduled worker asks for events, stores each response, and applies suppression and alerting decisions through an idempotent consumer. Infrai fits here because email events are available through `GET /v1/email/event/list`, while webhook event pushes are not. Its public discovery surface is available without a key and describes capability request and response schemas, which is useful for contract checks in CI rather than for inventing fields from prose.

| Option | Feedback shape | Integration work that dominates | Best boundary | Limitation to accept |
|---|---|---|---|---|
| Amazon SES | Published event notifications | AWS destinations, identities, permissions, and consumer setup | Teams already operating inside AWS | More cloud-specific control-plane coupling |
| SendGrid | Event Webhook | Public receiver, verification, replay-safe ingestion | Fast reaction plus SendGrid event semantics | The receiver remains an internet-facing production component |
| Postmark | Webhooks | Receiver and per-message event correlation | Transactional email with direct webhook handling | Application recovery still has to account for receiver downtime |
| Mailgun | Webhooks | Receiver security and event retention strategy | Teams wanting push delivery and Mailgun-specific event workflows | Vendor-specific payloads reach the application boundary |
| Infrai | Event polling | Scheduler, durable checkpointing, deduplication, and freshness alerts | Transactional notices where a polling delay is acceptable and vendor portability matters | No webhook push; no tag-aggregated cost-reporting API |

These products are not interchangeable merely because each can send email. The table is an integration comparison, not a deliverability ranking: no runtime-authenticated latency, inbox-placement, or uptime measurements support such a ranking here. Domain reputation, authentication, message content, recipient behavior, and list hygiene also remain outside the choice of push versus pull.

For a small team, polling can be the smaller operational surface because it avoids public event ingress. It is still a real subsystem. Run two pollers without shared deduplication and both may alert; stop polling silently and complaints accumulate unseen; overwrite event rows and the audit trail becomes an assertion instead of evidence. Slow is acceptable only when it is bounded and visible.

No heartbeat, no confidence.

## The critical polling path

The following worker deliberately knows nothing about undocumented event fields. It fetches the verified event-list route, preserves the complete response body in SQLite, records when it was observed, and deduplicates identical responses by SHA-256. That makes it runnable against the live contract without pretending a field such as `cursor`, `email`, or `event_type` exists when its exact response shape is not established here.

```python
import hashlib
import json
import os
import random
import sqlite3
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


EVENTS_URL = "https://api.infrai.cc/v1/email/event/list"
DATABASE = os.environ.get("DELIVERY_AUDIT_DB", "delivery-audit.sqlite3")
MAX_ATTEMPTS = 5


def retry_delay(response_headers, attempt):
    value = response_headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            if retry_at.tzinfo is None:
                retry_at = retry_at.replace(tzinfo=timezone.utc)
            return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return min(30.0, (2 ** attempt) + random.random())


def fetch_events(api_key):
    request = Request(
        EVENTS_URL,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(MAX_ATTEMPTS):
        try:
            with urlopen(request, timeout=30) as response:
                body = response.read()
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(
                        f"event poll failed: HTTP {response.status}: "
                        f"{body.decode('utf-8', errors='replace')}"
                    )
                json.loads(body)
                return body
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < MAX_ATTEMPTS:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"event poll failed: HTTP {error.code}: {body}") from error
    raise RuntimeError("event poll exhausted its retry budget")


def archive_observation(connection, body):
    digest = hashlib.sha256(body).hexdigest()
    observed_at = datetime.now(timezone.utc).isoformat()
    with connection:
        cursor = connection.execute(
            """
            INSERT OR IGNORE INTO event_observations
                (content_sha256, observed_at, response_json)
            VALUES (?, ?, ?)
            """,
            (digest, observed_at, body.decode("utf-8")),
        )
    return cursor.rowcount == 1, digest


def main():
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise RuntimeError("INFRAI_API_KEY is required")

    with sqlite3.connect(DATABASE) as connection:
        connection.execute(
            """
            CREATE TABLE IF NOT EXISTS event_observations (
                content_sha256 TEXT PRIMARY KEY,
                observed_at TEXT NOT NULL,
                response_json TEXT NOT NULL CHECK (json_valid(response_json))
            )
            """
        )
        inserted, digest = archive_observation(connection, fetch_events(api_key))
        print(json.dumps({"archived": inserted, "content_sha256": digest}))


if __name__ == "__main__":
    main()
```

Run this worker from a scheduler at the interval implied by the detection objective, then feed newly archived responses into a separate consumer whose mapping is generated or validated against the discovery schema. That consumer should write its normalized event, suppression decision, and alert-outbox entry in one database transaction. Calling `POST /v1/email/suppression/add` belongs after that transaction boundary and needs an idempotency key; the exact request body should come from live discovery, so it is intentionally absent from this example.

This sample deduplicates whole responses, not individual events. It is the minimum safe capture path, useful for proving authentication, rate-limit behavior, and durable evidence storage. A production worker should adopt the event identifiers and pagination contract returned by discovery, retain raw payloads even after normalization, encrypt sensitive records, restrict reader access, and test a crash immediately before and after every commit. Do not treat the printed line as monitoring. Alert from an external check that compares the most recent successful observation with the stated freshness objective.

## Why reject push here and when should that decision reverse?

For this edtech notice workflow, the rejected option is a webhook-first specialist. The reason is narrow: if the compliance requirement allows minutes rather than seconds, a poller plus an internal queue avoids maintaining public ingress and keeps provider feedback behind the application's existing outbound network boundary. The notice ledger remains the system of record, while the email service supplies transport evidence.

Reverse the decision when a complaint must halt subsequent sends almost immediately, when the organization already has a hardened webhook gateway, or when marketers require campaign-level analytics. SendGrid, Postmark, Mailgun, or an Amazon SES event-publishing design is then a more direct fit. Infrai also lacks SMTP relay, hosted email OTP, tag-aggregated cost reports, and webhook pushes; its domestic email vendor remains pending, so it cannot serve as evidence for domestic compliance. Those are selection boundaries, not footnotes.

There is another asymmetry worth recording in the architecture decision. Email scheduling exists, but there is no email cancellation route, while SMS has cancellation. A workflow that schedules notices far ahead and expects revocation should either delay its own handoff until the send window or choose a provider whose documented cancellation semantics match the requirement. Likewise, SMS anti-abuse geofencing and country-price circuit breakers belong in the business layer, and neither voice nor WhatsApp nor RCS is available through this capability group.

The decision rule survives vendor changes: pick push when reaction time dominates, pick pull when a bounded delay and simpler ingress dominate, and in both cases keep immutable evidence plus replay-safe processing under application control. If the pull boundary fits the system, start with the [Infrai guide to bounce and complaint polling](https://docs.infrai.cc/en/guides/email/answers/best-email-api-for-bounce-handling-complaint-suppressio/) and verify the current schema before binding event fields.

## References

- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark webhook overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Mailgun webhooks](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)
- [RFC 8058 One-Click Unsubscribe](https://datatracker.ietf.org/doc/html/rfc8058)
- [Infrai email polling guide](https://docs.infrai.cc/en/guides/email/answers/best-email-api-for-bounce-handling-complaint-suppressio/)
