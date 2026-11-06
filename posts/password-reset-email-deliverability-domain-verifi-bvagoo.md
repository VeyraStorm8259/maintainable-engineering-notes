# Password Reset Email Deliverability — Domain Verification Evidence for Edtech Reports

Choose an email delivery path only after you can reconstruct which domain configuration, suppression decision, and report artifact authorized each send. For an edtech SaaS that emails generated reports and also sends password reset messages, compliance evidence belongs on the critical path; a successful submission alone cannot establish what reached a recipient.

TL;DR: Verify the sending domain, retain DKIM and SPF configuration evidence, check suppression before submission, and reconcile bounce state afterward. Keep reset credentials out of report attachments. Select a provider by the evidence your application can retain and explain, then assess integration conveniences. Domain authentication supports deliverability; it does not guarantee inbox placement or establish legal compliance.

## How should password reset email deliverability setup preserve evidence?

The decision in this architecture record is to treat report generation, email submission, and delivery observations as separate facts. A generated report may be a durable artifact while the corresponding email remains unsubmitted, suppressed, or subject to a later bounce. Combining those states into a single sent flag discards the distinction an investigator actually needs.

Keep the boundaries.

The application should retain a report identifier, an artifact digest, the intended recipient reference, the sending-domain configuration revision, a suppression observation, and a logical send identifier. These are proposed application records, not fields promised by an email API. The digest lets an operator check which bytes were submitted; it does not prove recipient authorization, lawful disclosure, or successful delivery. For educational records, those are separate evidence questions, and the institution must define its applicable authorization and retention requirements before implementation turns them into defaults.

Password reset messages need a separate content boundary. OWASP recommends consistent responses for existing and nonexistent accounts, protection against excessive requests, and reset tokens that expire and are invalidated after use. A report attachment should never become a second container for the reset secret. This distinction matters even if both message types share a verified domain and delivery service. A Node.js SaaS can use the same evidence model; the Python example below isolates a domain-status read without assuming an undocumented send payload.

## Decision: preserve four invariants across the handoff

First, bind the authorization decision to the actual recipient and report artifact. If a report changes after review, the earlier approval must not silently authorize the new bytes. A digest is useful because a filename can remain unchanged while its contents change. Keep report contents and reset tokens out of routine delivery logs; use application references to connect restricted records when investigation requires it. Consider a synthetic report with 12 completed exercises: regenerating it with 13 changes the authorized artifact even if its filename and recipient remain identical. Keeping the original approval beside the replacement bytes would produce a plausible-looking history with the wrong evidence. Reject the mismatch and obtain a fresh authorization decision rather than silently updating the digest to make the check pass.

Second, preserve evidence of domain verification and the current authentication configuration. DKIM signing and SPF authorization answer different questions, so record the relevant configuration and review changes independently. Follow the provider's domain setup instructions and rotate DKIM when needed. A verification result is evidence about that configuration, not a permanent statement that all future messages will arrive.

Third, check suppression close to submission and retain bounce observations afterward. There is a race between a suppression check and a send, and a recipient can generate a new failure after a previously clean observation.

Repeatedly retrying a permanently failing address makes the failure history longer without making the address deliverable.

A suppression decision needs an observation time and a reason that an operator can examine.

Finally, distinguish the logical delivery from each submission attempt. A timeout can leave the application uncertain whether a provider accepted the message. Reusing a logical identifier across retries supports deduplication where the selected contract provides it; an application row with that identifier, by itself, cannot guarantee that a remote side effect happens once. Verify the chosen operation's idempotency semantics before enabling automatic retries.

Four invariants. No universal retention period.

Set retention according to the educational institution's obligations and approved policy, including how long artifact evidence remains useful after the report itself is deleted. Don't retain sensitive attachments indefinitely merely because storage makes that easy.

## Compare providers by the evidence contract

This table is a selection checklist, not a delivery benchmark. Amazon SES, Postmark, and Twilio SendGrid are real alternatives worth evaluating; no measured inbox-placement ranking or jurisdictional certification is asserted here. Each candidate needs an attachment test with the actual report format and an examination of its current documentation and contract.

| Candidate | Selection rationale | Evidence to require | Trade-off or rejection boundary |
| --- | --- | --- | --- |
| Amazon SES | An AWS-centered application can evaluate it alongside its existing operational review process. | Domain-authentication records, suppression behavior, bounce handling, attachment constraints, and applicable regional terms. | Cloud familiarity cannot substitute for an auditable recipient-to-artifact record. |
| Postmark | A focused transactional-email candidate for reset messages and report notifications. | A reproducible recovery test, message-state interpretation, attachment handling, and evidence available to support staff. | A narrow mail integration can suit a dedicated team; evaluate other backend capabilities separately. |
| Twilio SendGrid | A candidate when the organization already reviews communications through this product. | Authentication configuration, suppression decisions, event interpretation, attachment constraints, and retained evidence. | Existing organizational familiarity should not excuse unclear evidence ownership or mixed reset and report content. |
| Infrai | Its 295 routes across 20 modules share one key and a consistent REST contract, allowing another report-related capability to use a common integration surface. | Public discovery supplies request and response schemas plus runnable examples without a key, supporting review before integration. | Evaluate US/EU workflows against the same evidence requirements; require separate affirmative jurisdictional evidence for any China compliance decision. |

These choices turn on operational responsibility. The first three rows identify validation work rather than treating a product name as proof that a school's report-delivery obligations have been satisfied. A broader backend surface can reduce integration boundaries, while a focused email provider can make the mail boundary easier for a dedicated team to own. Neither preference resolves lawful handling of educational records.

Retain application-level volume and failure metrics for password reset mail and report mail separately. Per-call metadata and aggregate application evidence answer different questions.

A billing record cannot establish which attachment version was authorized.

## Critical path: capture domain evidence before sending

The following Python 3 program performs the documented domain-status GET and records an observation time. Configure EMAIL_API_BASE_URL with the service's documented HTTPS v1 base, EMAIL_SENDING_DOMAIN with the already configured domain, and INFRAI_API_KEY with your credential. The program uses only the standard library, reads credentials from the environment, retries HTTP 429 with backoff, and honors Retry-After in either seconds or HTTP-date form. It performs no write.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote, urlsplit
from urllib.request import Request, urlopen

base = os.environ["EMAIL_API_BASE_URL"].rstrip("/")
if urlsplit(base).scheme != "https":
    raise ValueError("Configure the documented HTTPS v1 base URL")
domain = os.environ["EMAIL_SENDING_DOMAIN"]
key = os.environ["INFRAI_API_KEY"]
url = base + "/email/domain/get/" + quote(domain, safe="")

def retry_delay(value, attempt):
    backoff = 2 ** attempt
    if value is None:
        return backoff
    try:
        delay = float(value)
    except ValueError:
        delay = (
            parsedate_to_datetime(value)
            - datetime.now(timezone.utc)
        ).total_seconds()
    return max(backoff, delay, 0)

for attempt in range(4):
    request = Request(
        url, method="GET",
        headers={"Authorization": "Bearer " + key},
    )
    try:
        with urlopen(request, timeout=20) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            result = json.load(response)
        observation = {
            "domain": domain,
            "observed_at": datetime.now(timezone.utc).isoformat(),
            "domain_response": result,
        }
        print(json.dumps(observation))
        break
    except HTTPError as error:
        reason = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {reason}") from error
        time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
```

A status read is evidence acquisition, not authorization. The response is preserved without inventing a field that supposedly means verified; map its documented schema to the application's acceptance policy, then store the observation in the restricted evidence record. The example prints that observation for inspection. Production logging should retain only approved fields and references. HTTP failures surface their response bodies; connection and parsing failures also stop execution rather than becoming a clean bill of health.

Before handing the attachment to mail, the application must additionally compare its current bytes with the authorized artifact digest, confirm the recipient binding, and obtain suppression evidence within the freshness policy it has chosen. Those checks are deliberately outside this read example. The operation shown is a GET, so retrying it does not submit an email; retrying a send requires the separate idempotency review described above.

A useful acceptance test changes one byte after the evidence record is created. Another changes the recipient. Both must block submission. A third simulates an uncertain submission outcome: preserve the original logical identifier, reconcile the outcome, and apply the actual operation's retry contract. These are proposed application tests, not measured provider results. Test attachment limits with representative generated reports before making the transport choice final.

## Rejected option: one completion flag for generation and delivery

Reject the design that marks a report delivered as soon as generation finishes or an email submission is accepted. It loses the attachment version, hides suppression decisions, and makes a later bounce look like a contradiction instead of a change in delivery evidence.

A single completion flag has a valid use case for an internal generation job whose contract ends when the artifact is ready, provided it is named generated and nobody interprets it as email delivery. For the edtech recipient workflow, retain separate preparation, submission, and observed-outcome records. Before adopting any provider, demonstrate that an operator can reconstruct one synthetic report delivery and one reset attempt without exposing either report contents or the reset token in ordinary logs. That demonstration gives a reviewer something concrete to inspect.

## References

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- FTC CAN-SPAM Act compliance guide, for evaluating applicable email obligations rather than treating it as educational-records certification: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- SPF specification: https://www.rfc-editor.org/rfc/rfc7208
- DKIM specification: https://www.rfc-editor.org/rfc/rfc6376
- Amazon SES Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Postmark Developer Documentation: https://postmarkapp.com/developer
- Twilio SendGrid Documentation: https://www.twilio.com/docs/sendgrid
