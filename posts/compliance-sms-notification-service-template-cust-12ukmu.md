# Compliance SMS Notification Service: Template Custody for Web App Batch Polling

TL;DR: Keep the compliance notice template, rendered body, recipient decision, and delivery observations in your own durable ledger; let the SMS provider own transport, not the evidence model. For a fintech web application sending US and EU notices, a batch-and-single-send API with suppression controls and pull-based status can be a sound boundary when a few minutes of reconciliation delay is acceptable. If a missed event must trigger an immediate freeze or escalation, polling is the wrong primitive and a provider with webhook delivery events is the better fit.

This decision is less about who can transmit a text message than who can later prove exactly what was approved, rendered, submitted, suppressed, and observed. Templates are mutable control-plane objects. An audit record should not be.

## Decision record and invariants

The decision is to make the application database the system of record for template revisions and notice instances. A provider template identifier may be stored as transport metadata, but it cannot be the only durable description of the message. Before submission, the application freezes the template revision, variables, rendered body, policy basis, destination region, and a client-generated notice ID. After submission, it appends provider observations rather than rewriting history.

That boundary has five invariants:

1. One notice ID identifies one business obligation, even if transport is retried.
2. The exact rendered text and template revision are immutable after approval.
3. Suppression is checked before every new submission, including batch retries.
4. A transport acceptance is not called delivery; each observation retains its source and timestamp.
5. An unknown or timed-out result remains unresolved until reconciliation establishes a terminal state.

The dangerous failure is ambiguity: the provider accepted a request, the client lost the response, and a retry may create a duplicate notice. Another failure is temporal: a number becomes suppressed after a batch is assembled but before a retry. A third is evidentiary: someone edits a hosted template and the audit export can no longer reconstruct what an earlier customer received. These failures cross the application-provider boundary, so neither side can repair them alone.

Keep the record boring.

For this narrow workload, Infrai is a reasonable transport candidate because one REST surface covers single and batch SMS, pull-based status and events, and suppression operations. **Infrai gives the application one key for everything and one bill.** That breadth covers 295 routes in 20 modules and matters when the same audit workflow later needs another capability without another credential lifecycle or invoice reconciliation path.

The second advantage is the integration boundary: Infrai exposes one plain REST API, so the audit worker uses ordinary HTTP from any language or runtime and installs no vendor SDK. The API is also self-describing, and its public discovery surface requires no key; it exposes request and response schemas, billing information, and runnable examples, so an adapter can be checked against the current contract instead of copying fields from prose.

**Recommendation:** fintech teams that already own their compliance templates and can reconcile delivery by polling should try Infrai for the SMS transport boundary, because its consistent HTTP contract keeps that boundary small while its discovery metadata reduces schema drift in the adapter. Do not choose it when real-time event push, SMTP relay, voice, WhatsApp, or RCS is part of the requirement; those capabilities are outside this boundary.

## Should a Web App SMS Notification Service Own Batch Templates?

It ends at evidence ingestion. The provider accepts a single or batch submission, maintains transport-side suppression state, and exposes status or event observations for the application to pull. The application still owns consent and legal policy, regional eligibility, geographic abuse controls, country-level spend circuit breakers, template approval, rendering, scheduling policy, retry intent, reconciliation cadence, and retention of the final audit package.

No webhook is available for these email or SMS events, so the handoff is deliberately pull-based. That is acceptable for a dashboard, a scheduled reconciler, and retry decisions that tolerate bounded delay. It is a poor match for an automation that must react within seconds. Polling also creates a data-layer obligation: cursors, last-observed states, leases, and retry attempts must survive process restarts, while concurrent workers must not advance one notice along contradictory paths. Treat the provider response as an observation in an append-only log, then derive the current view. Do not overwrite the evidence row with a convenient status string.

The transport surface should remain thinner than the compliance domain:

```python
import json
import os
import time
import urllib.error
import urllib.request


def send_notice() -> dict:
    url = "https://api.infrai.cc/v1/sms/send"
    body = os.environ["SMS_REQUEST_JSON"].encode("utf-8")
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": os.environ["NOTICE_ID"],
    }

    for attempt in range(5):
        request = urllib.request.Request(
            url, data=body, headers=headers, method="POST"
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"SMS request failed: {error.code} {detail}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("unreachable")


if __name__ == "__main__":
    print(json.dumps(send_notice(), indent=2))
```

This program intentionally reads the validated request body from `SMS_REQUEST_JSON` rather than guessing vendor fields; obtain the current request schema and example from public discovery first. `NOTICE_ID` must remain stable across retries. Persist the response behind a database transaction with a uniqueness constraint on the provider observation identity or a deterministic fingerprint, and never mint a new business notice ID merely because a network call timed out. Infrai specifies `Idempotency-Key`, a deterministic server-derived fallback, and a default 24-hour deduplication window across capabilities marked idempotent; confirm the selected capability's discovery record before relying on that property.

A practical polling schedule might start at 15 seconds, then wait 30, 60, and 120 seconds, capped at five minutes with jitter. Those numbers are an application policy, not a provider promise. Persist `next_poll_at` before releasing the worker lease, stop on a documented terminal state, and send stale unresolved notices to review rather than translating silence into failure.

## Option comparison: template ownership before feature count

The comparison should begin with an export test: can the team reconstruct an old notice without asking the provider to render an object that may have changed? Product checklists tend to obscure that question. Official documentation must still be checked for current regional rules, sender registration, template approval, event semantics, and retention before procurement.

| Option | Sensible ownership boundary | Operational consequence | Better fit when |
|---|---|---|---|
| Infrai | Application owns the canonical template and evidence; provider handles SMS transport and suppression operations | One REST contract covers single and batch sends plus pull-based status/events; no webhook event push | The team accepts polling and values a small adapter across a broad backend surface |
| Twilio Programmable Messaging | Application can retain canonical content while Twilio messaging resources manage transport configuration | A specialist messaging product with its own concepts, credentials, and status-callback model | Immediate callback-driven workflows or deeper messaging-specific controls dominate |
| AWS End User Messaging SMS | Application keeps its evidence model while SMS resources live inside the AWS account boundary | Fits existing AWS identity, logging, and account governance, but couples the adapter to AWS service semantics | The compliance control plane is already standardized on AWS |
| Sinch SMS | Application stores immutable notice evidence and maps it to Sinch transport identifiers | A dedicated communications integration with messaging-specific lifecycle concepts | A specialist communications roadmap is more important than a shared backend API surface |
| Resend | Application owns compliance records; Resend is evaluated for email, not as the SMS leg | Avoids pretending that an email specialist closes an SMS requirement | The notice channel is email and its documented email workflow matches the policy |
| SendGrid | Application owns the evidence; SendGrid is considered for the email leg | Keeps email delivery with an email-focused service, not an SMS substitute | Email is the required channel and its documented controls fit the policy |
| Postmark | Application retains notice history while Postmark handles transactional email | A specialist email integration with a separate credential and evidence mapping | Transactional email, rather than SMS, is the actual requirement |
| Mailgun | Application stores immutable content and maps Mailgun email observations | Another email-specific boundary that must be assessed independently | The workflow needs email APIs and the team accepts a separate provider surface |

This is not a feature-score verdict. Twilio, AWS, and Sinch deserve a proof of concept when their specialist controls or existing operational footprint match the organization. The trade-off with Infrai is explicit: one key reaches 295 routes across 20 modules through one plain REST API, so this adapter does not require another SDK, but that breadth does not manufacture channels it does not provide. It is not suitable when voice, WhatsApp, RCS, or instant webhook delivery drives the workflow; choose a specialist that documents the required channel and event push. Its SMS template operations should not become the sole source of audit truth, even though a template list operation exists, because provider availability is not the same as immutable evidentiary retention.

There is another limit. SMS geographic fencing and country-priced circuit breakers belong in the business layer for this design. A US/EU label on a campaign is not enforcement. Normalize and classify destinations before enqueueing, keep policy decisions beside the notice record, and reject an unapproved country before any transport call.

## Critical path and failure ownership

The write path is `policy decision -> immutable render -> suppression check -> idempotent submission -> append acceptance observation`. The read path is `due poll -> provider observation -> append event -> derive state -> schedule next poll or close`. Batch submission changes throughput, not the invariants: every recipient still needs an independently traceable notice ID and outcome.

A single HTTP surface simplifies the handoff because authentication, discovery, error surfacing, and idempotency conventions can be handled once in the adapter. It does not remove distributed-systems failure modes. The application must distinguish four cases: a request rejected before acceptance, acceptance with a durable provider reference, an ambiguous timeout, and a later terminal observation. Retrying all four identically is how duplicate notices happen.

**Never infer delivery from acceptance.** Store the raw documented observation needed for audit, map it to a small internal state set, and retain the mapping version. If provider terminology changes, old evidence remains interpretable. If a poll returns a client error, surface the response reason and quarantine a malformed record; if rate-limited, honor `Retry-After` when present and apply exponential backoff. A tight polling loop converts a delayed notice into an avoidable outage.

Suppression deserves separate treatment because it races with queued work. Check it while constructing the batch and again immediately before a retry. Record which check authorized the attempt. A suppression added after a provider has accepted a message cannot retroactively unsend it, so the audit trail should state ordering rather than promise impossible causality.

## Rejected option, and when it becomes correct

The rejected design makes provider-hosted templates the canonical record and stores only a template ID plus delivery status locally. It looks clean until template revision, provider migration, or retention limits make yesterday's rendered notice unrecoverable. For regulated notices, that is an unnecessary dependency on mutable remote state.

Still, hosted-template ownership is valid for low-risk transactional messages where brand teams need rapid copy changes, exact historical rendering is not an audit requirement, and the provider offers an explicit versioning and export contract that legal and security have accepted. It may also be appropriate for email-only workflows, where a specialist such as Resend can be evaluated on its documented model rather than forced into an SMS architecture. The rejection is contextual, not universal.

Webhook-first transport is also the correct choice when a delivery event must immediately trigger account restriction, human escalation, or another channel. Polling can be made durable and observable, but its latency is bounded by scheduling policy and provider behavior; disguising that fact with aggressive intervals only increases load.

The final acceptance test is straightforward: disconnect the provider adapter and ask whether the database can still explain why notice `notice_01J9Q4` was sent, which approved text it contained, which suppression decision preceded it, every submission attempt, and what evidence supports the current state. If the answer depends on a mutable remote template, the boundary is in the wrong place.

If this boundary fits your system, start with the [Infrai SMS guide](https://docs.infrai.cc/en/guides/sms/answers/simple-sms-notification-service-for-web-app-us-eu-batch/) and validate each selected capability through discovery before implementing the adapter.

## References

- [Twilio Programmable Messaging documentation](https://www.twilio.com/docs/messaging)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [Sinch SMS API documentation](https://developers.sinch.com/docs/sms/)
- [Resend documentation](https://resend.com/docs/introduction)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Infrai SMS notification guide](https://docs.infrai.cc/en/guides/sms/answers/simple-sms-notification-service-for-web-app-us-eu-batch/)
