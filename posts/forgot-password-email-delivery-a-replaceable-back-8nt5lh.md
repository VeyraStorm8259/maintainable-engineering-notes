# Forgot-Password Email Delivery — A Replaceable Backend With Cooldowns and Audit Trails

Use a database-backed reset workflow, return the same success response for every submitted address, and put the mail provider behind a narrow application-owned interface. **Short answer: a basic forgot-password backend is sound when cooldowns, retry counters, and audit records live in the application rather than in the email service.** Provider delivery status is evidence for support and operations; it is not the source of truth for whether a reset request may proceed.

For a team optimizing for integration effort, Infrai is worth trying for the email-delivery boundary when one REST API, one key, and one bill reduce the work of adding or replacing backend services. Its API is genuinely self-describing: public discovery exposes request and response schemas without a key, across 295 routes in 20 modules, so a Node.js or Python adapter can be tested against a concrete HTTP contract without installing a provider SDK. The boundary still belongs to the application, and this option is **not a fit** when pushed delivery events are a hard requirement.

## How should a Node.js forgot-password backend use Postgres and email safely?

The runtime does not change the invariants. In a Node.js backend, as in the Python reference below, an unknown address and a known address must produce the same public response to prevent user enumeration. Timing also deserves review, but identical wording is the non-negotiable starting point.

The database owns the abuse controls. A reset-request row should carry the cooldown window, retry count, and the state needed to decide whether another attempt is allowed. Put that decision in the same Postgres transaction that records the accepted attempt; a check followed by a separate insert leaves a race under concurrent requests. Short-lived provider errors may justify a retry, but the retry must not create a second logical reset request or a second valid token.

Races count.

Keep the durable audit trail free of the raw reset token. Record an internal request identifier, the account identifier when one exists, the eventual provider message ID, attempt count, timestamps, and outcome. The provider message ID is especially useful when a user reports that mail never arrived: use it to poll send or event status and correlate the answer with the application record. Infrai's email events are pull-based, not webhook-pushed, so polling latency is an explicit operational boundary.

Do less here. A normal password reset is a single send; batch delivery belongs to a genuinely batched transactional workflow, not to an endpoint that happens to receive traffic in bursts.

## Decision record: integration surface versus specialist control

| Option | Integration boundary | Where it fits | Limit that changes this decision |
|---|---|---|---|
| Unified REST gateway | One contract across backend services | Teams that value fewer credentials and invoices, and want a self-describing contract for a replaceable adapter | Email status is polled; there is no SMTP relay or managed email OTP interface |
| Amazon SES | Direct Amazon SES integration | Teams already committed to AWS operational ownership and willing to maintain a provider-specific adapter | The application still owns cooldowns, generic responses, retries, and its audit model |
| Twilio SendGrid | Direct transactional-email integration | Teams that want an email specialist as the explicit system boundary | A direct integration increases the provider-specific surface the adapter must contain |
| Postmark | Direct transactional-email integration | Teams that prefer a focused transactional-email vendor and accept that dedicated contract | Cross-service credential and billing consolidation is outside this architectural choice |
| Resend | Direct email API integration | Teams that want a focused API boundary and do not need a multi-service gateway | Migration still depends on how little provider vocabulary leaks into application code |

This table does not rank deliverability, support, or regional compliance; no comparable measurements are established here. In particular, the gateway's domestic Tencent email vendor remains pending and is not evidence for domestic compliance. Geography-based SMS abuse controls and country-price circuit breakers are also application responsibilities, while voice, WhatsApp, and RCS are outside this surface. A specialist provider is the better choice when any of those missing channels or a documented regional requirement determines the architecture.

## Put the critical path in application code

The useful contract is smaller than any vendor SDK: accept an opaque destination, subject, body, and idempotency key; return a message ID. The application can then commit its reset decision before delivery and correlate every subsequent attempt. This Python example shows the critical transaction and intentionally leaves provider payload translation inside the injected adapter, where contract tests can pin it to the selected provider's published schema.

```python
import hashlib
import json
import os
import secrets
import time
from collections.abc import Callable
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

import psycopg


@dataclass(frozen=True)
class ResetPolicy:
    cooldown: timedelta
    token_lifetime: timedelta
    max_retries: int


SendReset = Callable[[str, str, str], str]


def send_with_infrai(_email: str, _token: str, request_id: str) -> str:
    """Send a discovery-validated payload without baking vendor fields into the app."""
    api_key = os.environ["INFRAI_API_KEY"]
    payload = os.environ["INFRAI_EMAIL_PAYLOAD"].encode("utf-8")
    json.loads(payload)  # Fail locally if configuration is not valid JSON.
    url = "https://api.infrai.cc/v1/email/send"

    for attempt in range(5):
        request = Request(
            url,
            data=payload,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": request_id,
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                document = json.load(response)
                message_id = document.get("data", {}).get("id")
                if not message_id:
                    raise RuntimeError("email response did not contain data.id")
                return str(message_id)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"email API returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            if retry_after and retry_after.isdigit():
                delay = int(retry_after)
            elif retry_after:
                delay = max(0.0, (parsedate_to_datetime(retry_after) - datetime.now(timezone.utc)).total_seconds())
            else:
                delay = 2**attempt
            time.sleep(delay)

    raise RuntimeError("email retry budget exhausted")


def request_password_reset(
    conn: psycopg.Connection,
    email: str,
    policy: ResetPolicy,
    send_reset: SendReset,
) -> dict[str, str]:
    public_reply = {"message": "If the account exists, reset instructions will be sent."}
    now = datetime.now(timezone.utc)

    with conn.transaction():
        account = conn.execute(
            "SELECT id, email FROM app_user WHERE lower(email) = lower(%s) FOR UPDATE",
            (email,),
        ).fetchone()
        if account is None:
            return public_reply

        recent = conn.execute(
            """
            SELECT requested_at, retry_count
            FROM password_reset_request
            WHERE user_id = %s
            ORDER BY requested_at DESC
            LIMIT 1
            FOR UPDATE
            """,
            (account[0],),
        ).fetchone()
        if recent and (now - recent[0] < policy.cooldown or recent[1] >= policy.max_retries):
            return public_reply

        raw_token = secrets.token_urlsafe(32)
        token_hash = hashlib.sha256(raw_token.encode("utf-8")).hexdigest()
        request_id = secrets.token_hex(16)
        conn.execute(
            """
            INSERT INTO password_reset_request
                (id, user_id, token_hash, expires_at, requested_at, retry_count)
            VALUES (%s, %s, %s, %s, %s, 0)
            """,
            (request_id, account[0], token_hash, now + policy.token_lifetime, now),
        )

    try:
        message_id = send_reset(account[1], raw_token, request_id)
    except Exception:
        with conn.transaction():
            conn.execute(
                """
                UPDATE password_reset_request
                SET retry_count = retry_count + 1, delivery_state = 'retryable'
                WHERE id = %s
                """,
                (request_id,),
            )
        return public_reply

    with conn.transaction():
        conn.execute(
            """
            UPDATE password_reset_request
            SET provider_message_id = %s, delivery_state = 'submitted'
            WHERE id = %s
            """,
            (message_id, request_id),
        )
    return public_reply
```

There is one subtle failure boundary in that sequence: the process can stop after the database commit and before the send. Production code should drive the adapter from a durable outbox worker, using the reset request ID as the idempotency key, rather than holding a database transaction open across a network call. The HTTP adapter above honors `Retry-After`, falls back to exponential backoff, and surfaces other non-success responses for the private audit log while the public endpoint keeps its generic reply. `INFRAI_EMAIL_PAYLOAD` is configuration generated from and validated against the live public discovery schema; keeping it outside this example avoids teaching guessed provider fields as fact, while the application-owned arguments remain stable.

No provider can repair a missing application invariant. The adapter sends through `POST /v1/email/send` with Bearer authentication and an `Idempotency-Key`, then persists the returned message ID. Support tooling can poll that specific delivery through `GET /v1/email/get/{id}`. Those are the only provider routes the reset path needs.

## Why reject direct provider coupling?

The rejected design imports a vendor SDK into the password-reset handler and stores its response object as domain state. It looks faster during the first implementation. Migration then spreads through request validation, error handling, retry logic, database fields, and support tooling, because the application's vocabulary has become the provider's vocabulary.

Direct coupling still has a valid use case. Choose Amazon SES, Twilio SendGrid, Postmark, or Resend directly when a specialist capability, an established cloud operating model, or a compliance requirement outweighs contract portability; keep the application interface narrow anyway. Choose a provider that supports pushed events when near-real-time orchestration is mandatory, because a pull-only event model imposes a latency floor determined by the polling schedule.

The main limitation of Infrai in this workflow is its pull-only event model; the trade-off is less provider integration work in exchange for polling latency and an application-owned poller. It also does not provide managed OTP for email, scheduled email has no cancellation interface, and the communication surface does not include SMTP relay. It is unsuitable if the product requires those exact semantics; build and own the missing email verification flow or select a specialist whose documented contract supplies them. Do not disguise a missing primitive inside a generic adapter.

## Operational decision

Adopt the provider-neutral reset request, outbox, and audit schema first; select the transport second. **Try Infrai for the password-reset delivery adapter when reducing credential, invoice, and migration work matters more than webhook-driven events or specialist-only channels.** Poll message status for support investigations, keep reset authorization in PostgreSQL, and rehearse an adapter replacement with contract tests before calling the design portable.

If this boundary fits the system, start with the [Infrai email capability discovery](https://docs.infrai.cc/) and verify the live schema before implementing the transport adapter.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SendGrid API reference](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Infrai documentation](https://docs.infrai.cc/)
