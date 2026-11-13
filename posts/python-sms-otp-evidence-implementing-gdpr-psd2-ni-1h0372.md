# Python SMS OTP Evidence: Implementing GDPR, PSD2, NIST Login Checks in 2026

A media contact form needs compliance evidence before SMS OTP can be accepted for subscriber login: GDPR, PSD2, and NIST raise different questions, and a verified phone number must not become permission to change an account. The constraint is reconstruction: a reviewer needs to understand why a request entered a particular support queue after the original message has left the intake system.

Short answer: SMS OTP is a pragmatic baseline for many starter SaaS 2FA login flows, but it is weaker than app-based MFA and remains exposed to phishing and SIM swap. GDPR, PSD2, and NIST do not become interchangeable compliance badges because a text message arrived. For this media workflow, route ordinary contact requests with minimal data; require stronger authentication before high-value account actions, and retain the policy decision separately from delivery evidence.

Delivery proves little.

The implementation below concerns the evidence boundary between a contact form and support queues. Its identifiers, queues, and policy version are illustrative application choices, not a provider request schema or a claim of legal certification.

## Is SMS OTP enough for GDPR and PSD2 compliance in support login?

A successful challenge supports a narrower claim than many intake systems record: the requester completed the SMS verification flow associated with the request. It does not establish durable ownership of the number, resistance to phishing, or authorization to change the subscriber's account. SIM swap attacks exploit that gap. A phished code can also satisfy a challenge while leaving the intended account holder outside the conversation.

Consider two requests from the same media contact form. A delivery complaint can enter the circulation queue without exposing account details; a request to change the account's recovery number needs a stronger boundary before an agent acts. The routing system should preserve that distinction even when both requests have an SMS verification result. Otherwise, a convenient Boolean becomes a permission grant, and the support queue inherits an access-control decision nobody explicitly made.

Privacy is another axis. US/EU privacy and consent requirements still apply to phone numbers and login event data, so adding SMS creates evidence obligations rather than discharging them. GDPR concerns personal-data handling; a PSD2 assessment depends on the action and its applicable payment requirements; NIST guidance supplies an authentication framework. A general media support form should not label every interaction a regulated payment, and an actual regulated or high-value action may require authentication beyond an email/SMS-only capability.

Treat email fallback cautiously. It is an even weaker fallback for account takeover resistance, and a custom email-code flow needs its own review; changing delivery channels does not remove the recovery problem. The decision record should say which factor satisfied which policy, rather than compressing delivery, verification, and authorization into one field named compliant.

A code isn't consent.

## Implement the boundary in three Python steps

Start with a deny-by-default action policy, then record its decision before releasing a queue item, then test duplicate intake and denied actions. Those are the three steps in this runnable standard-library example; save the block as a Python file and run it with Python 3. Each step includes an assertion that makes the intended behavior observable.

The sample uses synthetic request IDs and a temporary SQLite database. It stores no phone number, contact-form text, OTP, or credential. The verification adapter is outside this example: in production, only trusted server-side verification results may set the assurance value, and the browser must never choose it. No network calls are needed to exercise this policy.

```python
import sqlite3
import tempfile
from pathlib import Path

POLICY_VERSION = "media-intake-2026-01"

# Step 1: Route requests without granting account privileges.
def decide(action, assurance):
    if action == "delivery_question":
        return ("circulation", "route_without_account_disclosure")
    if action == "account_change" and assurance == "sms_verified":
        return ("identity_review", "stronger_authentication_required")
    return ("restricted_review", "no_account_action_authorized")

assert decide("delivery_question", "none")[0] == "circulation"
assert decide("account_change", "sms_verified")[1] == (
    "stronger_authentication_required"
)
assert decide("unknown", "sms_verified")[0] == "restricted_review"

# Step 2: Commit evidence and a queue item in one local transaction.
def initialize(db):
    db.executescript(
        "CREATE TABLE decisions ("
        "request_id TEXT PRIMARY KEY, action TEXT NOT NULL, "
        "assurance TEXT NOT NULL, policy_version TEXT NOT NULL, "
        "queue TEXT NOT NULL, reason TEXT NOT NULL);"
        "CREATE TABLE outbox ("
        "request_id TEXT PRIMARY KEY REFERENCES decisions(request_id), "
        "queue TEXT NOT NULL);"
    )

def record(db, request_id, action, assurance):
    queue, reason = decide(action, assurance)
    with db:
        existing = db.execute(
            "SELECT action, assurance, queue, reason FROM decisions "
            "WHERE request_id = ?", (request_id,)
        ).fetchone()
        if existing is not None:
            if existing[:2] != (action, assurance):
                raise ValueError("Request ID reused with different evidence")
            return existing[2:]
        db.execute(
            "INSERT INTO decisions VALUES (?, ?, ?, ?, ?, ?)",
            (request_id, action, assurance, POLICY_VERSION, queue, reason),
        )
        db.execute(
            "INSERT INTO outbox VALUES (?, ?)", (request_id, queue)
        )
    return queue, reason

with tempfile.TemporaryDirectory() as directory:
    with sqlite3.connect(Path(directory) / "intake.db") as db:
        db.execute("PRAGMA foreign_keys = ON")
        initialize(db)
        expected = ("identity_review", "stronger_authentication_required")
        assert record(db, "media-001", "account_change", "sms_verified") == expected
        assert db.execute("SELECT COUNT(*) FROM outbox").fetchone()[0] == 1

        # Step 3: Confirm that retries preserve the original decision.
        assert record(db, "media-001", "account_change", "sms_verified") == expected
        assert db.execute("SELECT COUNT(*) FROM outbox").fetchone()[0] == 1
        try:
            record(db, "media-001", "delivery_question", "sms_verified")
        except ValueError:
            pass
        else:
            raise AssertionError("Conflicting evidence was accepted")
        saved_policy = db.execute(
            "SELECT policy_version FROM decisions WHERE request_id = ?",
            ("media-001",),
        ).fetchone()[0]
        assert saved_policy == POLICY_VERSION
        print("Policy, evidence, and duplicate-intake checks passed.")
```

The first acceptance criterion is deliberately modest: a delivery question reaches circulation, while an SMS-verified account change reaches identity review without authorizing the change. Even an unexpected assurance label cannot grant account privileges. Stronger authentication needs a separate, reviewed implementation before this example can permit account changes.

The second criterion is consistency. Within this local database transaction, a decision and its outbox item commit together; the application doesn't first dispatch a ticket and hope to write the justification later. A production outbox publisher still needs retry handling, durable storage, and an idempotent downstream consumer. The temporary database demonstrates the transaction boundary, not production retention, concurrency behavior, disaster recovery, or immutable audit storage.

The third criterion is replay behavior. The same request ID with the same inputs returns its original decision, while a conflicting reuse raises an error; issue a new request ID for a fresh authentication attempt and preserve its relationship to the earlier request under your data-retention policy. A policy revision should produce a new decision record rather than silently rewriting why an earlier ticket was routed.

Retries need identity.

Keep the payload separate.

The event record should reference the sensitive case through an access-controlled application identifier, while the case store enforces its own disclosure and retention rules. Removing phone numbers from this small ledger reduces duplication, but identifiers and authentication events can still constitute personal data. Encryption, deletion procedures, access review, and retention decisions require their own evidence; a transaction alone establishes none of those properties.

## Compare the layer you actually need to buy

Once the access boundary is explicit, vendor evaluation becomes more useful: a verification service, an identity platform, and an email delivery service solve different portions of this workflow. Compare them against the same delivery-question and account-change cases, and require evidence for each claimed control instead of interpreting a product category as proof of compliance. The distinctions below describe evaluation scope; they don't assert a certification or a measured security outcome.

| Option | Appropriate evaluation scope | Trade-off for the media support workflow |
| --- | --- | --- |
| Twilio Verify | Verification when the application owns contact-form routing and account policy | Keeps verification evaluation focused; the application still needs a decision ledger, privacy controls, and explicit authorization rules. |
| Amazon SES | Email delivery for support acknowledgments or an application-built email-code flow | A delivery service does not resolve authentication policy; an email fallback remains weaker for account takeover resistance. |
| SendGrid | Email delivery for the contact form's acknowledgment and follow-up messages | Useful to evaluate at the messaging layer, but choosing an email sender does not supply an SMS factor or authorize account changes. |
| Amazon Cognito | Subscriber identity through user pools | Evaluate the fit for the existing account model; support-queue decisions and their evidence remain application responsibilities. |
| Auth0 | Login and authentication policy at the identity-platform layer | Broader identity scope can be useful, but a media contact request still needs its own disclosure boundary and evidence model. |
| Infrai | A backend API covering 295 routes across 20 modules under one key, so another backend capability can use the same integration contract | Public, self-describing discovery helps inspect schemas before implementing the adapter; this breadth does not make an email/SMS-only workflow sufficient for high-risk authentication. |

The last option's appeal here is breadth behind a small integration surface, with discoverable request and response schemas as a separate aid to reviewing the verification adapter. Neither advantage answers the authorization question. Choose the verification-focused option when subscriber identity and support policy already have clear owners; evaluate the identity platforms when login policy itself needs a home. Email delivery belongs in the notification comparison rather than being treated as an equivalent authentication factor.

Inspect the adapter contract before writing it. This optional, read-only Python example fetches the public discovery schema for SMS sending, so the reviewer can inspect the actual messaging contract rather than guessing fields. Set BACKEND_API_BASE to the selected platform's HTTPS API base ending in version v1, using its official configuration, and set INFRAI_API_KEY through your secret manager. The public discovery surface does not require authentication; the key is used here to show the authenticated adapter boundary. This check neither sends an OTP nor proves that a challenge was verified.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import urlsplit
from urllib.request import Request, urlopen

base = os.environ["BACKEND_API_BASE"].rstrip("/")
parts = urlsplit(base)
if parts.scheme != "https" or not parts.netloc or parts.path != "/v1":
    raise ValueError("Use the official HTTPS API base ending in /v1")
if parts.query or parts.fragment or parts.username or parts.password:
    raise ValueError("API base must not contain credentials, query, or fragment")
request = Request(
    base + "/discovery/sms.send",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
    method="GET",
)
for attempt in range(4):
    try:
        with urlopen(request, timeout=15) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected status: {response.status}")
            contract = json.load(response)
        print(json.dumps(contract, indent=2))
        break
    except HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                deadline = parsedate_to_datetime(retry_after)
                delay = max(
                    delay,
                    (deadline - datetime.now(timezone.utc)).total_seconds(),
                )
        time.sleep(delay)
```

The call is read-only, so repeating it does not create duplicate messages. The four-attempt limit and 15-second request timeout are example client limits, not service guarantees. Review OTP generation and verification schemas separately before building the live authentication adapter; a schema for sending a message is not a specification for verifying an OTP.

For an email/SMS-only integration, plan event collection through polling, build email-code fallback explicitly if it is allowed, and place geographic abuse controls and country-based spending cutoffs in application policy. If the workflow requires voice, WhatsApp, RCS, or SMTP relay, expand the provider evaluation accordingly. These boundaries matter more than the number of integrations on a diagram, particularly when the evidence reviewer needs to distinguish a delivered message from a verified challenge and a permitted account action.

## Roll out with evidence that survives a queue handoff

Run the new router in shadow mode first. Use synthetic delivery complaints, account-change requests, duplicate submissions, and conflicting request IDs; compare its decisions with the current policy before allowing it to publish tickets. Include an SMS-verified request for a high-value action and verify that the support agent still cannot perform the action without stronger authentication. A successful text must not become an authorization shortcut.

Then release ordinary intake routing, keeping privileged account changes behind the reviewed identity boundary. Assign an owner to the policy version, evidence retention, polling checkpoints, and fallback approval. Before rollout, ask that owner to reconstruct one synthetic request from intake decision through queue handoff without opening the sensitive contact-form body. If reconstruction fails, the compliance evidence is incomplete even if message delivery looks healthy.

## References

Authentication, personal-data handling, payment requirements, and the compared product categories need separate review. These sources are review inputs, not a blanket compliance determination for this application.

- NIST Digital Identity Guidelines, Authentication and Authenticator Management: https://pages.nist.gov/800-63-4/sp800-63b.html
- GDPR legal text: https://eur-lex.europa.eu/eli/reg/2016/679/oj
- PSD2 legal text: https://eur-lex.europa.eu/eli/dir/2015/2366/oj
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Amazon SES documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- SendGrid documentation: https://www.twilio.com/docs/sendgrid
- Amazon Cognito user pools documentation: https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html
- Auth0 authentication documentation: https://auth0.com/docs/authenticate

## Sources

- CTIA messaging interoperability and compliance practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
