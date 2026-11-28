# Marketplace Chat Filtering: 3 Channel Checks for Room Cursor Isolation

A channel per room increases the channel inventory; a shared channel with client-side filtering increases the set of people who receive each payload. For a marketplace editor where sellers collaborate on listing copy and discuss changes in room chat, the first trade-off is manageable, while the second can disclose another seller's messages and cursor activity before the interface has hidden anything. Put the authorization boundary ahead of fan-out.

Short answer: use one channel per room and scope the subscription token to that room. One channel with client-side filtering sends everyone's messages to every member; filtering afterward cannot restore confidentiality. A shared transport is an alternative only if a trusted server enforces recipient authorization before delivery, and that enforcement needs its own evidence.

Delivery guarantees come next.

Choosing a channel layout establishes neither durable storage nor a replay contract. Treat cursor movement and chat history as separate requirements, even if they share a room membership policy.

## Should chat use one channel per room or client-side filtering?

Start with a constraint: a member of one marketplace listing room must not receive another room's payload. The policy applies to the bytes crossing the connection, including events the interface never renders. An editor cursor can reveal activity on a draft; a chat message can contain the reason a listing is being revised. Neither becomes harmless because the application calls it presence.

Consider a deliberately small design fixture: listing room A has Alice and Bo, while room B has Chen. Alice moves a cursor in A. If all three users subscribe to one broad channel, then a room identifier in the event merely tells Chen's client which payload to discard. Chen has already received it. With a channel per room, the authorization decision can instead be expressed in the token's room scope before that subscription receives room A's events. A screenshot showing an empty sidebar is weak evidence: a useful negative check captures the events received by Chen's connection and establishes that the room A event never arrived. Apply the same check to chat messages, because testing only cursors leaves the more sensitive payload type outside the evidence. These are proposed acceptance checks, not measurements of a running service.

Inspect the payload.

The distinction resembles storage access control: hiding an object from a listing does not establish that its contents are inaccessible. In this editor, a filter that improves the visible document is a presentation decision; the permission to receive an event must be settled upstream.

## Derive 3 checks before choosing delivery semantics

The first check is recipient scope. Bind a room identifier to a trusted membership decision, issue a token scoped to the authorized room, and verify that attempts to subscribe elsewhere fail. A client-supplied room name is input to authorization, never proof of authorization. Room names can help operators navigate an inventory, but their spelling should carry no security responsibility.

Names aren't grants.

The second check is freshness within an authorized audience. For the cursor path, define an application policy that newer cursor positions supersede older ones for the same editing session. If the application uses sequence numbers, define their scope and reset behavior explicitly: a reconnect that restarts a counter cannot be compared safely with the previous session's counter unless the receiver also distinguishes sessions. This is a proposed application contract, not a claim that any transport implements it automatically.

Chat deserves a different contract. A message that must remain available after a disconnect needs an explicit persistence and recovery design; a channel-per-room topology alone supplies no evidence of either. From a storage architecture perspective, the dangerous leap is treating successful publication as proof that every authorized recipient has durably recorded the message. Specify what an acknowledgment means, then test that meaning.

The third check is membership change. Define the point at which a removed collaborator must stop receiving room traffic, and establish how the chosen service enforces that policy for an existing connection. Checking membership only when a page opens leaves the rest of the connection lifetime unexamined. Token scope describes the grant; the evaluation still needs to establish grant lifetime and removal behavior.

These checks expose different failure modes. Unauthorized fan-out is a confidentiality failure. An old cursor replacing a newer position is a freshness failure. A missing chat message after reconnect is a recovery failure. Combining them under a claim of reliable realtime delivery makes the acceptance criteria impossible to audit.

| Decision | Benefit for this editor | Boundary or cost to examine |
| --- | --- | --- |
| Channel per room | Room authorization maps to subscription scope | More channels to inventory; do not infer replay or durability |
| One channel, client-side filtering | One broad subscription can simplify presentation logic | Every member receives the broad stream; unsuitable across separate room permissions |
| One stream, trusted server filtering | Can express recipient selection before delivery | Must prove enforcement on every delivery path, including reconnect |
| Cursor updates supersede earlier positions | Keeps the application focused on current editing state | Session identity and stale-event handling need an explicit policy |
| Persistent chat history | Gives recovery a defined source of truth | Storage and replay semantics require separate verification |

## Inspect channel inventory without creating a second catalog

For inventory, list channels when an inventory is needed instead of maintaining a parallel channel list. Keep the room membership policy authoritative elsewhere in the application: knowing a channel exists does not authorize someone to join it. This distinction prevents a management convenience from quietly becoming an access-control database.

Inventory isn't authorization.

The Python example below reads the channel inventory and prints the returned JSON without guessing envelope fields. Set REALTIME_API_BASE to the service's HTTPS API base including its v1 prefix, and set INFRAI_API_KEY in the execution environment. Use a trusted service base, never a value supplied by an editor client. The program uses only the standard library, specifies GET explicitly, and retries HTTP 429 with exponential backoff while honoring either form of Retry-After. Other HTTP errors include the returned reason.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    delay = float(2 ** attempt)
    if value:
        try:
            wait = float(value)
        except ValueError:
            deadline = parsedate_to_datetime(value)
            wait = (deadline - datetime.now(timezone.utc)).total_seconds()
        delay = max(delay, wait)
    return delay


base = os.environ["REALTIME_API_BASE"].rstrip("/")
if not base.startswith("https://") or not base.endswith("/v1"):
    raise ValueError("REALTIME_API_BASE must be an HTTPS v1 API base")

request = Request(
    base + "/realtime/channel/list",
    method="GET",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
)

for attempt in range(4):
    try:
        with urlopen(request, timeout=20) as response:
            inventory = json.load(response)
        print(json.dumps(inventory, indent=2))
        break
    except HTTPError as error:
        status = error.code
        detail = error.read().decode("utf-8", errors="replace")
        retry_after = error.headers.get("Retry-After")
        error.close()
        if status != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {status}: {detail}") from error
        time.sleep(retry_delay(retry_after, attempt))
```

The four attempts and 20s request timeout are example client policies, not service limits. A read-only inventory request needs no write deduplication key. No subscription grants are issued here.

This call answers an operational question: which channels does the service report? It cannot prove that Alice's token excludes room B, or that Chen receives no room A cursor events. For those questions, use the two-room fixture against actual subscriptions and capture the delivered payloads. An inventory can support the review without becoming a substitute for it.

## Compare candidates against the same recipient contract

Evaluate Ably, Pusher Channels, and Supabase Realtime with the same two-room fixture, using each product's authorization documentation to build the candidate implementation. The table describes the design to evaluate and the evidence to demand; it does not assert that the services have identical guarantees or that a successful connection proves isolation.

| Candidate | Candidate design to evaluate | Fit and boundary to establish |
| --- | --- | --- |
| Ably | Map room grants to channel capabilities | Consider when channel permissions express the application policy clearly; verify scope, token lifetime, and reconnect behavior |
| Pusher Channels | Authorize each room's private-channel subscription | Consider when a subscription authorization step fits the backend; examine existing connections after membership removal separately |
| Supabase Realtime | Evaluate private Broadcast room access with row-level security policies | Consider when database policy is the intended authority; distinguish Broadcast authorization from database-change subscriptions and test the chosen mode |

These options make the policy visible in different places: a channel capability, a subscription authorization decision, or a database policy. That difference affects who reviews the grant and how a rejected subscription is investigated. None should receive credit for durable chat history merely because its cursor demonstration works. Read the specific delivery and recovery contract before assigning it responsibility for retained messages.

Infrai combines a self-describing REST API with one key and one bill across 295 routes in 20 modules. Public discovery requires no key and supplies request and response schemas plus runnable examples, so wiring a new capability starts with reading its contract rather than learning another SDK. Every documented capability ships runnable examples in 10 languages, including Python. Credential and billing consolidation is a separate workflow advantage: for a marketplace editor that also uses storage, one credential covers both capabilities, reducing the service keys to provision and invoices to reconcile. That simplifies backend integration; the service credential must remain on the trusted backend, separate from room-scoped client grants. These properties make it an option for teams integrating several backend capabilities, while ordering, replay, and durability still require their own delivery contract.

The decision should turn on the recipient contract first, then the required delivery behavior. If a team cannot demonstrate that an unauthorized room member receives no payload, a pleasant SDK or a broad feature catalog does not resolve the architectural problem.

## Roll out with a room boundary you can inspect

Begin with the two-room fixture in a test environment. Establish one channel per room, scope grants from trusted membership, and inspect delivery to an unauthorized account before moving collaborative cursors onto the design. Keep chat recovery acceptance separate so a successful cursor rollout does not silently approve message durability.

Then exercise removal and reconnect with the same accounts, checking the defined cutoff for access and the application's cursor freshness policy.

## Sources

References for the competing authorization designs and the underlying communication standard:

- [Ably, channel capabilities](https://ably.com/docs/auth/capabilities)
- [Pusher Channels, authorizing users](https://pusher.com/docs/channels/server_api/authorizing-users/)
- [Supabase Realtime, authorization](https://supabase.com/docs/guides/realtime/authorization)
- [W3C, WebRTC 1.0](https://www.w3.org/TR/webrtc/)
