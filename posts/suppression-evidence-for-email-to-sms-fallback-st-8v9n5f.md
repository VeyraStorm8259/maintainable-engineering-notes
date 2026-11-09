# Suppression Evidence for Email to SMS Fallback Status Polling and Timeout Recovery

Require a suppression proof before email to SMS fallback releases marketplace event notifications to another channel. A buyer's bounced email address and delayed status polling demand different decisions: one can justify excluding an invalid recipient, while the other leaves an unresolved message that may still arrive.

TL;DR: For email to SMS fallback, let a scheduled worker poll status, retain the observations, and evaluate a timeout against current channel eligibility. Polling makes the handoff approximate. The useful compliance evidence is the record explaining why this recipient could receive this notice at dispatch time, including what was suppressed and what remained unknown.

This decision concerns marketplace order exception notices. Its focus is proof of recipient eligibility, rather than a promise of instantaneous escalation or exactly one notification across two channels.

## Decision record for a suppression proof

Adopt an application-owned decision record with two distinct inputs: delivery observations and recipient restrictions. Preserve both, but do not let one masquerade as the other. A delivery timeout says the worker's waiting period expired; it does not establish that an address is invalid, that the buyer wants a text, or that an earlier email will never arrive. Those distinctions matter because a permanent restriction can outlive the individual order notice that caused it, and a badly inferred restriction can silently exclude the buyer from later communications.

Unknown stays unknown.

The invariant is that each dispatch decision must refer to the recipient policy revision it evaluated. Within the application's transactional boundary, recheck that revision when creating dispatch intent, and retain the observation identifiers, decision time, timeout deadline, and reason. An application record should also identify the order notice and relevant authorization evidence without duplicating the buyer's contact details throughout the audit history. These are proposed application fields, not claims about a provider's response schema.

The failure boundary lies between that transaction and the external send. A database commit cannot atomically commit a remote delivery. Use a stable logical attempt identity, apply the selected sender's documented idempotency contract, and reconcile an ambiguous response before creating another send attempt. A successful HTTP exchange is also a different kind of evidence from delivery, and delivery is a different kind of evidence from human attention.

There are three named failure modes to reject during review. **Timeout poisoning** turns missing evidence into permanent suppression. A stale authorization snapshot releases a message after recipient policy changes. Cross-channel laundering treats an email failure as permission to contact the buyer by SMS. Each produces a superficially complete notification history with a missing justification at its center.

## What can delayed email status polling prove before SMS fallback?

A poll proves what the worker observed at a particular time, within the meaning of the documented response. Keep receipt time separate from any reported event time; otherwise an observation arriving late can appear to have informed a decision that was already made. Retain the original observation under the marketplace's access and retention policy, together with its normalized interpretation and the version of the mapping that produced it. Normalization is a policy boundary worth reviewing, especially where temporary delivery problems, permanent recipient failures, and complaints lead to different actions.

For a concrete example, choose a 300-second email waiting window and a nominal 60-second poll interval. These are illustrative marketplace settings, not service guarantees. A worker that notices the deadline on its next run can authorize fallback later than the deadline; scheduler delay and delayed observations can widen that gap. If a late email arrives after an SMS is released, retain both outcomes. Rewriting the history into a clean, exclusive handoff would destroy evidence rather than improve delivery.

A complaint needs its own policy treatment. It is not merely an invalid mailbox.

Don't infer consent.

Keep open tracking outside the eligibility proof. Apple's Mail Privacy Protection changes what senders can learn from email activity, and neither an open signal nor a delivery status proves that a buyer read and understood an order exception. The fallback decision should cite a defined timeout or qualifying outcome, independent SMS authorization, and the absence of a current restriction for the contemplated send.

For this polling architecture, both channel histories are pulled rather than pushed as webhook events. Treat scheduling and reconciliation as application responsibilities. Keep escalation within email and SMS; requirements involving voice, WhatsApp, or RCS call for a different channel plan. If email verification is part of the order workflow, define its code lifecycle in the application rather than assuming a managed email OTP facility. Sender eligibility, regional compliance evidence, geographic restrictions, and country-based SMS spending circuit breakers also need explicit owners. A route name is not compliance evidence.

## Compare observation contracts before choosing an adapter

The acceptance test is a buyer restriction arriving between enqueue and dispatch. Ask each candidate to support a trace showing the original observation, the restriction it caused, and the final eligibility check; a polished delivery dashboard cannot answer that question by itself.

| Candidate | Relevant observation contract | Fit for the marketplace decision | Evidence still owned by the application |
| --- | --- | --- | --- |
| Amazon SES | Email event publishing through configuration sets | Suitable to evaluate when email events will feed the recipient record | The mapping from a bounce to suppression and the subsequent SMS authorization |
| Twilio SendGrid | Email Event Webhook | Suitable to evaluate when email observations should enter through event ingestion | Duplicate observation handling, suppression scope, and the policy used at dispatch |
| Twilio Programmable Messaging | Outbound message status callbacks | Suitable to evaluate for the SMS part of a two-channel design | Correlation with the email attempt and evidence that SMS contact was permitted |
| Shared REST option | Plain REST API callable with ordinary HTTP requests, without an SDK to install | Suitable for an application-owned polling worker whose email and SMS adapters share credentials | Observation retention, timeout policy, suppression decisions, and authorization for each channel |

These are different integration surfaces, not four interchangeable end-to-end orchestration products. The email candidates need an SMS arrangement, while the SMS candidate needs an email arrangement. Event ingestion can reduce dependence on a periodic poll, but incoming observations still need durable handling and current policy checks. The REST option is attractive when the marketplace accepts approximate fallback timing and wants to inspect schemas before building the adapter; that convenience does not establish the meaning of a bounce or the buyer's permission to receive a text.

The trade-off is explicit: polling is not suitable for an instant event-driven handoff. Choose SendGrid's Event Webhook for email event ingestion or Twilio's status callbacks for SMS observations when callback delivery is a requirement, while retaining the same recipient policy boundary. Public discovery with request and response schemas helps implement the polling adapter, but doesn't eliminate that timing limitation.

Evidence takes work.

Infrai, the shared REST option, offers a plain REST API with no SDK required, a single key and unified billing across 295 routes / 20 modules. One API key and one bill reduce credential and invoice coordination between the email observer and SMS dispatcher; they don't merge delivery outcomes. Its self-describing public discovery exposes request and response JSON Schema without a key, allowing adapter review before credentials enter the deployment.

Cancellation deserves a separate acceptance test. Keep notices under application scheduling until dispatch if cancellation before submission is an invariant, and evaluate channel-specific cancellation behavior independently. An explicit SMS cancel operation does not establish a dedicated scheduled-email cancellation workflow, and neither should be treated as a universal recall promise. Similarly, confirm regional sending arrangements with affirmative evidence; API availability alone cannot establish local compliance.

## Critical path from an observation to a dispatch decision

The example below pairs a read-only email status request with a local transactional demonstration: a permanent email failure suppresses email, an unresolved outcome leaves email eligibility unchanged, and a newer SMS restriction defeats a fallback decision. It runs with Python's standard library and SQLite. All normalized event names and fields are application-defined; a real adapter must map documented provider outcomes into them without guessing.

The two synthetic notices use a 300-second waiting window. The code records dispatch intent, not an external send. Set NOTIFICATION_API_ORIGIN to the HTTPS service origin without a version path, set INFRAI_API_KEY, and pass an existing email message ID as the first script argument. The real response is saved whole to marketplace_observations.sqlite3; the synthetic decisions remain separate, because inventing a delivery-status field would corrupt the evidence boundary.

```python
import email.utils
import json
import os
import sqlite3
import sys
import time
import urllib.error
import urllib.parse
import urllib.request


def fetch_observation(message_id):
    origin = os.environ["NOTIFICATION_API_ORIGIN"].rstrip("/")
    if not origin.startswith("https://"):
        raise ValueError("The service origin must use HTTPS")
    path = "/v1/email/get/{id}".format(
        id=urllib.parse.quote(message_id, safe="")
    )
    request = urllib.request.Request(
        origin + path, method="GET",
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
                 "Accept": "application/json"},
    )
    for attempt in range(4):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                raw = response.read().decode("utf-8")
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"HTTP {response.status}: {raw}")
                json.loads(raw)
                return response.status, raw
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = float(2 ** attempt)
            value = error.headers.get("Retry-After")
            if value:
                try:
                    required = float(value)
                except ValueError:
                    required = (
                        email.utils.parsedate_to_datetime(value).timestamp()
                        - time.time()
                    )
                delay = max(delay, required)
            time.sleep(delay)
    raise RuntimeError("Polling attempts exhausted")


def observe(db, event_id, notice_id, outcome, observed_at):
    allowed = {"unknown", "delivered", "permanent_recipient_failure"}
    if outcome not in allowed:
        raise ValueError("An adapter must explicitly map this outcome")
    with db:
        inserted = db.execute(
            "INSERT OR IGNORE INTO observations VALUES (?, ?, ?, ?)",
            (event_id, notice_id, outcome, observed_at),
        ).rowcount
        if inserted and outcome == "permanent_recipient_failure":
            db.execute(
                "UPDATE policy SET email_blocked = 1, revision = revision + 1 "
                "WHERE recipient = (SELECT recipient FROM notices WHERE id = ?)",
                (notice_id,),
            )


def decide(db, notice_id, now):
    with db:
        db.execute("BEGIN IMMEDIATE")
        row = db.execute(
            "SELECT n.recipient, n.deadline, p.sms_allowed, p.sms_blocked, "
            "p.revision FROM notices n JOIN policy p "
            "ON p.recipient = n.recipient WHERE n.id = ?",
            (notice_id,),
        ).fetchone()
        if row is None:
            raise ValueError("Unknown notice")
        recipient, deadline, allowed, blocked, revision = row
        outcomes = set(
            value[0] for value in db.execute(
                "SELECT outcome FROM observations WHERE notice_id = ?",
                (notice_id,),
            )
        )
        if blocked or not allowed:
            reason = "sms_ineligible"
        elif "delivered" in outcomes:
            reason = "email_delivery_observed"
        elif now < deadline:
            reason = "waiting_for_email"
        else:
            reason = "deadline_elapsed_with_sms_authorization"
        db.execute(
            "INSERT INTO decisions VALUES (?, ?, ?, ?)",
            (notice_id, now, revision, reason),
        )
        if reason == "deadline_elapsed_with_sms_authorization":
            db.execute(
                "INSERT OR IGNORE INTO intents VALUES (?, ?, ?)",
                (notice_id, recipient, revision),
            )
        return reason


db = sqlite3.connect(":memory:")
db.executescript("""
CREATE TABLE policy (
    recipient TEXT PRIMARY KEY, email_blocked INTEGER NOT NULL,
    sms_allowed INTEGER NOT NULL, sms_blocked INTEGER NOT NULL,
    revision INTEGER NOT NULL
);
CREATE TABLE notices (
    id TEXT PRIMARY KEY, recipient TEXT NOT NULL, deadline INTEGER NOT NULL
);
CREATE TABLE observations (
    id TEXT PRIMARY KEY, notice_id TEXT NOT NULL,
    outcome TEXT NOT NULL, observed_at INTEGER NOT NULL
);
CREATE TABLE decisions (
    notice_id TEXT NOT NULL, decided_at INTEGER NOT NULL,
    revision INTEGER NOT NULL, reason TEXT NOT NULL
);
CREATE TABLE intents (
    notice_id TEXT PRIMARY KEY, recipient TEXT NOT NULL, revision INTEGER NOT NULL
);
INSERT INTO policy VALUES ('buyer-17', 0, 1, 0, 1);
INSERT INTO policy VALUES ('buyer-23', 0, 1, 0, 1);
INSERT INTO notices VALUES ('order-exception-41', 'buyer-17', 300);
INSERT INTO notices VALUES ('order-exception-42', 'buyer-23', 300);
""")

observe(db, "observation-1", "order-exception-41", "unknown", 120)
observe(db, "observation-2", "order-exception-42",
        "permanent_recipient_failure", 180)
observe(db, "observation-2", "order-exception-42",
        "permanent_recipient_failure", 180)
with db:
    db.execute(
        "UPDATE policy SET sms_blocked = 1, revision = revision + 1 "
        "WHERE recipient = ?", ("buyer-17",)
    )

print(decide(db, "order-exception-41", 360))
print(decide(db, "order-exception-42", 360))
print(db.execute(
    "SELECT recipient, email_blocked, revision FROM policy ORDER BY recipient"
).fetchall())
db.close()

message_id = sys.argv[1]
http_status, raw = fetch_observation(message_id)
with sqlite3.connect("marketplace_observations.sqlite3") as evidence:
    evidence.execute(
        "CREATE TABLE IF NOT EXISTS raw_evidence ("
        "message_id TEXT, observed_at REAL, http_status INTEGER, payload TEXT)"
    )
    evidence.execute(
        "INSERT INTO raw_evidence VALUES (?, ?, ?, ?)",
        (message_id, time.time(), http_status, raw),
    )
```

The first decision is SMS-ineligible because the recipient restriction changed before evaluation. The second creates one logical SMS intent after the deadline; replaying the same permanent-failure observation does not increment the policy revision again. Buyer 17's unresolved email remains unsuppressed. Buyer 23's confirmed permanent recipient failure produces an email restriction. The synthetic time values demonstrate ordering, not measured delivery performance.

The HTTP request has a 20-second timeout and at most four attempts, with exponential backoff for HTTP 429 and respect for Retry-After. Other errors propagate without changing recipient eligibility. A long server-requested delay can exceed a worker's budget; production scheduling should persist the next eligible poll time and release the worker rather than sleeping through that delay. This read-only request needs no write deduplication key.

The important limit is after intent creation. A production dispatcher must revalidate current eligibility when claiming an intent, preserve the intent's stable identity across retries, and retain enough evidence to reconcile an ambiguous external response. A restriction arriving after that final check may race an external send; the database cannot recall a message already submitted. Document that boundary instead of labeling the local transaction “exactly once.” The sample deliberately models a single email attempt per notice; production records also need attempt identities and observation provenance so an old attempt's result cannot be attached to a new one.

## Rejected option and the evidence required to revisit it

Reject a timer that sends SMS whenever email has not been marked delivered. It has a valid use case as a deliberately approximate reminder for recipients already authorized for both channels, where duplicate contact is acceptable and the workflow does not infer permanent suppression. It is insufficient for this marketplace job because silence cannot explain either recipient invalidity or the right to switch channels.

Revisit that choice only with a narrower requirement or stronger evidence. Acceptance should include an unresolved poll that leaves recipient eligibility intact, a replayed bounce that preserves one policy change, a restriction arriving before dispatch, and a late delivery observation retained after fallback. The resulting record must answer a practical question: what did the marketplace know, and which policy permitted this particular contact? It can support a compliance review; it does not replace one.

## References

- [Apple, Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES, event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html)
- [Twilio SendGrid, Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Twilio Programmable Messaging, outbound message status callbacks](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
