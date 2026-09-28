# Urgent Event Notifications: 4-Step Node.js SMS-First Email Fallback Delivery Logic

An urgent travel itinerary change is not delivered merely because an SMS gateway accepted it. The operational constraint is the departure clock: poll for useful delivery evidence only while SMS can still arrive in time, then send one email fallback without moving the event's original deadline. **Short answer:** persist a four-state dispatch record, schedule bounded delivery polling, and make the fallback transition atomic and idempotent. Never hold a Node.js application request open while polling.

This distinction matters in travel support. A passenger may receive another schedule change while the first message is delayed, and two independent workers may observe the same inconclusive status. The retry logic must tolerate both events without sending a cascade of notifications or quietly treating an obsolete itinerary as current. The useful unit is the itinerary revision, not the passenger address alone: a later revision can supersede an earlier one, while a retry of the same revision must retain the same identity.

## How should Node.js poll urgent SMS notifications before email fallback?

An initial provider response establishes submission, not handset receipt. SMS delivery is store-and-forward, and status evidence may remain provisional or arrive after the reset token has expired. Email has a similar boundary: SMTP defines transfer and reply semantics, while acceptance by a server does not prove that a person saw the message. The system therefore needs to record evidence, not translate every provider response into the word `delivered`.

Use four internal states: `sms_pending`, `sms_delivered`, `email_pending`, and `closed`. Provider-specific statuses belong in an append-only attempt log alongside the raw event time and external message identifier. The compact business state answers what the workflow may do next; the attempt log answers what happened. Keeping those concerns separate prevents a late SMS callback from overwriting the fact that email fallback has already started.

The deadline should be computed once from the itinerary event's creation time. For example, with a ten-minute action window, reserving the final two minutes for email means SMS observation stops at minute eight. Those numbers are policy inputs, not universal constants: teams should set them from measured channel latency and the minimum useful time a traveler needs to react. The invariant matters more than either number.

Fallback cannot extend `expires_at`.

## Model the transition, not the transport

The reliable unit is a database transaction around a dispatch row. A worker claims a due row, asks the adapter for current evidence, and applies one legal transition. If evidence is still inconclusive and the SMS observation deadline has not passed, it increments the poll count and schedules the next check with exponential delay. If the deadline has passed, it atomically changes the state to `email_pending`; a separate outbox worker sends email using a stable idempotency key.

That split is deliberate. Polling and sending in the same transaction would hold locks across network calls, while sending before committing the transition leaves a crash window in which a replacement worker can send the same fallback. A transactional outbox does not make the external channel exactly-once, but it makes local intent durable and gives the adapter a repeatable key. Duplicate suppression still depends on the receiving API honoring that key or on reconciliation against the stored external identifier.

Here is the core policy as transport-neutral Python. Database locking, the outbox insert, and the state update must occur in one transaction in the real implementation.

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum


class Evidence(Enum):
    DELIVERED = "delivered"
    TERMINAL_FAILURE = "terminal_failure"
    INCONCLUSIVE = "inconclusive"


@dataclass(frozen=True)
class Dispatch:
    dispatch_id: str
    expires_at: datetime
    sms_observe_until: datetime
    poll_count: int


def decide(dispatch: Dispatch, evidence: Evidence, now: datetime) -> dict:
    if now >= dispatch.expires_at:
        return {"next_state": "closed", "reason": "credential_expired"}

    if evidence is Evidence.DELIVERED:
        return {"next_state": "sms_delivered", "reason": "delivery_evidence"}

    fallback_due = (
        evidence is Evidence.TERMINAL_FAILURE
        or now >= dispatch.sms_observe_until
    )
    if fallback_due:
        return {
            "next_state": "email_pending",
            "outbox_key": f"{dispatch.dispatch_id}:email-fallback",
            "expires_at": dispatch.expires_at.isoformat(),
        }

    delay = min(15 * (2 ** dispatch.poll_count), 120)
    return {
        "next_state": "sms_pending",
        "poll_after_seconds": delay,
        "poll_count": dispatch.poll_count + 1,
    }
```

Do not treat every error alike. A timeout while reading status is inconclusive; an explicit terminal failure can justify immediate fallback; malformed credentials are an operational fault and should stop automated retries until configuration is repaired. Rate limiting calls for the server-provided retry guidance when available, bounded by `sms_observe_until`. This taxonomy prevents a transient observation failure from becoming a false delivery failure. It also exposes a real trade-off: a shorter polling window gives email more time but abandons potentially successful SMS delivery earlier, while a longer window preserves the preferred channel at the risk of making fallback useless for a near-term gate or platform change.

## Where reliability actually fails

The dangerous cases sit between systems, not inside the happy-path adapter. They deserve explicit tests.

| Failure mode | Required behavior | Evidence to retain |
|---|---|---|
| Worker crashes after claiming a poll | Lease expires; another worker resumes | Claim time, lease owner, poll count |
| Two workers cross the fallback boundary | One conditional update wins | Prior state, transition time, outbox key |
| SMS callback arrives after email starts | Append evidence; do not reverse state | Provider event time and receipt time |
| Status endpoint times out | Retry observation within the deadline | Error class, latency, next check |
| User requests a second reset | Revoke or supersede by explicit policy | Challenge generation and expiry |
| Email is accepted, then the response is lost | Reconcile by idempotency key or message ID | Attempt ID and external ID |

Clock handling is another quiet trap. Store UTC instants, compare them on the server, and never derive itinerary urgency from a handset timestamp. A monotonic clock is useful for measuring one process's elapsed call time, but persisted scheduling needs a shared wall-clock instant and monitoring for clock skew. Short action windows make even modest skew visible.

Observability should follow the state machine. Count transitions by reason, measure time from challenge creation to the strongest available channel evidence, and alert on age percentiles for `sms_pending` and `email_pending`. Also track duplicate outbox claims and late callbacks. Avoid putting phone numbers, email addresses, reset URLs, or codes into logs and metric labels; stable opaque dispatch identifiers are enough for correlation.

## Compare boundaries before adapters

A provider comparison should begin with failure semantics rather than a feature checklist. The relevant questions are whether status changes can be pushed, polled, or both; which states are terminal; how duplicate requests are handled; how retry guidance is exposed; and how long event records remain available for reconciliation. A polished SDK cannot compensate for ambiguous delivery evidence.

This architecture has limits. It is a poor fit for events with no meaningful response deadline, because polling adds load and state without improving the outcome; callback-only processing may be enough there. It is also insufficient for life-safety notices, where SMS-first plus email fallback provides too few independent paths and delivery evidence remains weaker than acknowledgement. The explicit state machine buys auditability and bounded work, but the cost is another durable record, an outbox, reconciliation, and operational ownership. That is the trade-off, not a universal recommendation.

| Decision axis | Stronger boundary | Risk requiring mitigation |
|---|---|---|
| Delivery evidence | Documented state meanings and timestamps | A handset receipt can still be unavailable |
| Retry control | Explicit rate-limit and retry signals | Retrying beyond expiry wastes capacity |
| Idempotency | Stable request key with defined scope | Keys may expire before reconciliation |
| Event delivery | Signed callbacks plus status lookup | Callbacks can be delayed, duplicated, or reordered |
| Data residency | Clear processing and retention controls | Message metadata may cross regions |

For US and EU recipients, geography should be data in the routing policy, not a branch scattered through application code. Store the recipient's region and consent basis separately from the dispatch, select an approved channel route, and keep retention short enough for the support purpose. Legal and carrier requirements vary by message category and destination, so engineering needs an owned policy review rather than a hard-coded assumption that a transactional reset message is exempt from every rule.

RFC 8058's one-click unsubscribe mechanism applies to list email and is not a delivery fallback protocol. Itinerary-change mail should remain narrowly transactional; mixing marketing material into it changes the compliance and user-expectation analysis. RFC 5321 is the useful technical boundary here because it defines SMTP transfer behavior without promising human attention.

## Roll out without betting the reset path

Start by recording decisions in shadow mode while the existing sender remains authoritative. Compare predicted fallback transitions with observed channel evidence, especially near the expiry boundary, and inspect disagreements without logging secrets. Then enable the state machine for a small, deterministic cohort, with a kill switch that returns control to the established path.

Test crashes at each durable boundary: before and after the conditional state update, before and after the outbox insert, and after the external send but before acknowledgement is stored. Also inject reordered callbacks and status timeouts. **The release criterion is not a high send-success percentage; it is the absence of illegal transitions and unbounded work under those failures.**

Once enabled broadly, retain the four-state record long enough to reconcile late evidence, then delete or aggregate it according to the support retention policy. The final architecture is intentionally plain: a deadline, a state row, an attempt log, a transactional outbox, and adapters that translate external evidence without owning business policy.

Keep it plain.

## Sources

- https://datatracker.ietf.org/doc/html/rfc5321
- https://datatracker.ietf.org/doc/html/rfc8058
- https://www.twilio.com/docs/sms
- https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE
