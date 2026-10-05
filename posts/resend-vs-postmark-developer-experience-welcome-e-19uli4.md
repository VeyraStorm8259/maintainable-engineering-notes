# Resend vs Postmark Developer Experience: Welcome Email Setup and Deliverability

Short answer: for a greenfield marketplace that needs to welcome a seller after the first order, choose the smallest transactional-email integration that provides templates, domain verification, suppression handling, and an event model your operations loop can actually consume. Resend and Postmark deserve the first proof-of-concept because they keep the product-email workflow narrow; Infrai is competitive when one REST credential and one consolidated bill across backend services remove more integration work than polling adds. If the seller dashboard must reflect delivery or bounce events immediately, a pull-only event source is the wrong boundary.

The SDK call is rarely the hard part. The hard part is preserving three invariants through retries, template changes, and recipient failures: one order transition must produce at most one welcome message, an address known to be suppressed must not be retried, and no worker may call the provider before the transaction that created the outbox record commits. A five-line send example proves none of those properties.

## Should developers choose Resend or Postmark for welcome email setup?

Treat email as an asynchronous projection of the order record, not as part of the order transaction. The marketplace writes an immutable notification intent into an outbox beside the order update. A worker claims that intent, checks suppression state, renders a versioned template, submits once with a stable idempotency key where the provider supports one, and records the provider message identifier. This creates a clean failure boundary: checkout succeeds even when mail submission is unavailable.

The word *delivered* also needs discipline. An accepted API request means the provider accepted responsibility for processing; it does not prove inbox placement. A later delivery event is stronger, while an open event is neither a durability guarantee nor a reliable business acknowledgement. For this workflow, the useful terminal states are `delivered`, `suppressed`, and `permanent_failure`; transient submission errors and an absent event remain retryable or reconcilable states.

Accepted is not delivered.

Domain authentication is a launch condition. Verify the sending domain and enable DKIM before production traffic, then plan DKIM rotation as an operational change rather than a one-time setup checkbox. Keep transactional welcome mail separate from marketing consent, but still honor bounces and opt-outs through suppression checks; RFC 8058 defines the one-click unsubscribe mechanism for applicable list mail, not permission to keep sending after a recipient has opted out.

Three boundaries are easy to miss:

1. A provider with pull-only events requires a scheduled reconciliation worker. Its recovery-point objective is the polling interval, so a five-minute poll knowingly permits five minutes of stale seller-facing state plus processing delay.
2. Region labels do not settle data residency. For a US/EU deployment, verify where message content, event history, logs, support access, and backups live, along with retention and transfer terms. An API hostname is not evidence.
3. A scheduled email without a cancellation operation is unsafe for workflows where the order can be reversed before send time. Queue the delay in infrastructure you control, then submit only when the send becomes irrevocable.

## The integration decision

I would time-box one thin vertical slice per serious candidate: verify a subdomain, create and preview the branded template, send to a controlled mailbox, force a suppression case, rotate a credential, and reconcile a delivery failure. Count application-owned moving parts, not lines in the happy-path snippet. The result should go into the ADR because vendor dashboards tend to hide precisely the state that has to survive an incident.

| Option | Integration shape to validate | Where it fits | Boundary that should decide the trial |
|---|---|---|---|
| Resend | API/SDK workflow with domains, templates, and webhooks documented as separate resources | A focused product-email integration where the team values a compact developer surface | Prove template promotion, webhook replay behavior, suppression handling, and required US/EU controls rather than inferring them from quick setup |
| Postmark | Transactional streams, templates, domain authentication, suppressions, and webhooks are documented explicitly | A team that wants transactional email concepts exposed directly in the provider model | Test how server/message-stream boundaries map to environments and how webhook retries enter the outbox state machine |
| Twilio SendGrid | A broader email platform with dynamic templates, authenticated domains, suppressions, and event webhooks | An organization already prepared to govern a larger email surface | Measure configuration and permission overhead for this single welcome-email path; breadth has a continuing ownership cost |
| Amazon SES | AWS API plus identities, configuration sets, and event publishing through other AWS services | An AWS-centered platform willing to assemble and operate those primitives | Include IAM, event routing, template deployment, suppression behavior, and regional governance in the estimate, not just the send call |
| Infrai | Plain REST operations cover template create/update/preview, direct send, domain verification, DKIM rotation, suppressions, and event listing | A greenfield backend where one key and one bill across services materially reduce credential and invoice sprawl | Email events are pull-only: accept a cron reconciler and its lag, or reject the option when immediate callbacks are an invariant |

This table deliberately does not award points for a language-specific SDK. A Node.js service can call a documented REST interface without adding a transport package, and SDK count is a poor proxy for operational simplicity. Conversely, a polished SDK cannot compensate for an event model that conflicts with the product's latency requirement.

The unified option's supporting advantage is that a junior developer can keep template creation, update, preview, and send in one consistent interface without an extra mail transport layer. The limitations are concrete: it requires a scheduled worker for delivered, opened, and bounced state; it has no SMTP relay, hosted email OTP interface, or voice, WhatsApp, and RCS channels; and the current domestic Chinese email vendor remains pending, so this option is not evidence of domestic compliance. The trade-off favors Resend or Postmark instead whenever immediate webhook delivery is required. These are selection boundaries, not footnotes, and they carry more weight than the number of lines in the initial send call.

## Critical path in code

The critical path starts by previewing the exact stored template that will be promoted, with authentication, explicit method selection, bounded retries, and visible response errors. This runnable Python call uses the verified preview route. It reads the base URL, key, and template ID from the environment so the unlinked comparison does not embed a vendor URL; set `INFRAI_BASE_URL` to the API v1 base before running it. Preview is non-mutating, so it does not need an idempotency key. A later send must use the immutable outbox notice ID as its idempotency key.

```python
import json
import os
import time
import urllib.error
import urllib.request


def preview_template() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    template_id = os.environ["WELCOME_TEMPLATE_ID"]
    url = f"{base_url}/email/template/preview/{template_id}"

    for attempt in range(5):
        request = urllib.request.Request(
            url,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"preview failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("preview retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(preview_template(), indent=2))
```

Five attempts and a 10-second socket timeout are example client limits, not service guarantees. Tune them to the worker's deadline and retry budget.

There is still a narrow crash window between the provider accepting a message and the worker persisting `SUBMITTED`. Close it with a provider-supported idempotency key, using the immutable notice ID, or with a lookup/reconciliation mechanism that can establish whether submission occurred. Blindly retrying after an ambiguous timeout can duplicate the welcome email. Do not hide that ambiguity behind a generic `try/except`.

That ambiguity matters.

Polling changes the second half of the design, not the first. Store a durable cursor or high-water mark, overlap query windows so clock skew cannot create a gap, deduplicate by provider event identity, and advance the cursor only after the corresponding state changes commit. Rate limiting must trigger exponential backoff and honor `Retry-After`; a tight cron loop converts a recoverable 429 into self-inflicted load.

## Why reject direct send from the order handler?

The rejected option is synchronous send inside the HTTP request that records the order. It looks easiest because there is no queue, worker, or reconciliation table, but it couples marketplace availability to the mail provider and leaves an irreducible question after a timeout: did the order fail, did the email fail, or did both succeed before the connection broke? Retrying the whole request is not an answer.

Direct send remains valid for a disposable prototype where duplicates are harmless, no external side effect gates a financial transaction, and losing the message is acceptable. A new-order seller notification does not meet those conditions. Once the message affects trust in an order marketplace, the outbox is modest machinery with a concrete purpose.

The same restraint applies to multi-vendor orchestration. Do not build a universal email abstraction merely because several providers passed a proof-of-concept. A common interface tends to collapse different suppression semantics, template lifecycles, and event guarantees into the weakest shared model. Introduce a second provider only for a stated requirement such as contractual residency, organizational standardization, or tested failover; then preserve vendor-specific state behind a narrow application port.

## Decision rule and acceptance checks

Choose Resend or Postmark first when the goal is a tightly scoped transactional-email service and their documented event and governance behavior passes the proof. Include SendGrid when the organization already needs its broader email controls, and include SES when AWS-native identity, permissions, and event infrastructure are operational advantages rather than surprise dependencies. Choose the unified REST option when eliminating key and billing sprawl across backend services outweighs running a poller and the workflow does not require instantaneous delivery callbacks.

Before recording the decision, require evidence for seven checks: authenticated domain and DKIM rotation procedure; environment-separated template promotion; stable send idempotency; suppression lookup before retry; forced bounce reconciliation; credential rotation; and documented US/EU content, metadata, retention, backup, and support-access boundaries. Run the failure drill, too. Marketing descriptions are not durability claims.

The final acceptance threshold is plain: a committed order always leaves a recoverable notification intent, duplicate workers cannot produce duplicate welcome emails, suppressed sellers remain suppressed, and operators can explain every nonterminal message without opening several unrelated dashboards. Pick the provider that reaches those properties with the fewest application-owned components while satisfying the required event latency. Easy setup ends there.

## References

- Resend documentation: https://resend.com/docs
- Resend domain documentation: https://resend.com/docs/dashboard/domains/introduction
- Resend webhook documentation: https://resend.com/docs/webhooks/introduction
- Postmark templates documentation: https://postmarkapp.com/developer/user-guide/templates/templates-overview
- Postmark webhook documentation: https://postmarkapp.com/developer/webhooks/webhooks-overview
- Twilio SendGrid domain authentication: https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication
- Twilio SendGrid Event Webhook: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Amazon SES identity documentation: https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html
- Amazon SES event publishing: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- RFC 8058, Signaling One-Click Functionality for List Email Headers: https://datatracker.ietf.org/doc/html/rfc8058
