# Nobody Knows Which Service Holds This API Key: 2026 Usage Debugging

When nobody knows which property-management service holds an API key, the least complex way to stop it from draining a prepaid balance is to recover the inventory, read usage per key, and deal with the inactive set first. Do that before rotating everything or tracing every deployment. The bill tells you which credentials matter; startup identity logging prevents the same ambiguity from returning.

**TL;DR:** rank keys by recent usage, assign the active keys to services, rename each key as soon as its owner is known, and treat a key with no recent usage as the safest revoke-and-observe candidate. Add credential identity to startup logs before cleanup begins. The critical design choice is blast radius: one shared credential may be convenient, but it turns one uncertain owner into many uncertain dependencies.

## How do you debug which service holds an API key?

A prepaid balance does not disappear because a key exists. It falls because calls charged to one or more live credentials continue to arrive. Start by expressing the bill as a distribution: for each credential, divide its usage by total usage during the same window. If one key accounts for 72% of observed usage, that is the dominant term; if eight keys each account for roughly one-eighth, the investigation has a very different shape. Those percentages are examples of the calculation, not measurements or vendor benchmarks.

This distinction matters in a property-management stack. Resident messaging, maintenance intake, lease document processing, and nightly reconciliation may run on different schedules, yet an unlabeled credential makes them look like one anonymous consumer. A system owner who starts with repository search can spend hours mapping configuration files while the balance keeps moving. Usage per key collapses the search space first.

No guesswork yet.

I would resist revoking the busiest key merely because it is the most visible one.

The first snapshot should be read-only and preserved as an audit artifact with a timestamp and the response bodies. The following script calls only the two verified account surfaces needed to recover inventory and usage. It deliberately writes the returned JSON without assuming undocumented field names; inspection and correlation happen against the actual response rather than a brittle parser built from guesses. That restraint can feel inconvenient during a live debug session, especially when a guessed `key_id` field would make a quick join easy, but a fabricated schema is worse than a manual first pass because it can silently assign spend to the wrong service.

```python
import json
import os
import sys
import time
from datetime import datetime, timezone
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = os.environ["ACCOUNT_API_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def get_json(path: str, attempts: int = 5):
    for attempt in range(attempts):
        request = Request(
            f"{BASE_URL}{path}",
            method="GET",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("request attempts exhausted")


snapshot = {
    "captured_at": datetime.now(timezone.utc).isoformat(),
    "keys": get_json("/account/keys/list"),
    "usage": get_json("/account/usage"),
}
json.dump(snapshot, sys.stdout, indent=2, sort_keys=True)
sys.stdout.write("\n")
```

Run it with the key supplied through the environment and redirect stdout into access-controlled audit storage. The credential value itself must never enter the artifact. OWASP's secrets-management guidance is blunt on this point: secrets need lifecycle controls, access restrictions, rotation, and auditability; an inventory dump containing the secret would defeat the investigation's purpose.

The number to calculate next is concentration. Rank usage by key, record the top key's fraction of the total, then record how many keys cover 90% of usage. This tells you where attention changes the bill fastest without pretending that volume proves ownership. A burst may come from a scheduled reconciliation job; a steady stream may belong to resident notifications. Usage narrows the suspects. It does not name them, recover deleted deployment history, or prove which process currently holds the key.

## Can an inactive credential be revoked safely?

Not with certainty. A key with no recent usage is the safest candidate for a revoke-and-see operation, but “recent” must cover the longest legitimate service interval. A monthly owner-statement job will look dead in a seven-day sample. A seasonal lease-renewal task may be quiet longer still. Choose the observation window from the workload schedule, not from impatience.

The cleanup order is therefore asymmetric. Active keys demand attribution before disruption. Inactive keys can enter a controlled revoke-and-observe queue, provided the team has named rollback authority, watches the affected property workflows, and records the decision in the audit trail. Revocation is an experiment with a bounded blast radius, not a substitute for inventory.

Shared keys deserve the most skepticism. If one credential is mounted into resident messaging, maintenance dispatch, and accounting, revoking it tests three workflows at once and the result says little about which service was responsible. A credential per deployment identity produces a smaller failure domain and a cleaner cost ledger. There is an operational price: more objects to govern, more rotation events, and more policy assignments. For systems funded from one prepaid balance, that overhead buys attribution.

| Credential pattern | Attribution quality | Blast radius on revoke | Operational cost | Best fit |
|---|---:|---:|---:|---|
| One key for the whole property platform | Low | All dependent workflows | Low | Short-lived prototypes with one owner |
| One key per environment | Medium | One environment | Medium | Small teams with clear staging and production boundaries |
| One key per service and environment | High | One workload boundary | Higher | Production systems where spend and incident ownership must be traceable |
| Short-lived workload identity | High | One identity policy | Highest migration effort | Platforms already able to issue and renew ephemeral credentials |

The table is not an argument for maximum granularity everywhere. A tiny internal job with the same owner, release cycle, and risk boundary as its parent service may reasonably share that service's key. Split credentials where independent revocation or attribution would change an incident decision.

## Build the audit trail before changing the inventory

Startup identity logging belongs before cleanup because every restart during the investigation can then produce new ownership evidence. Log the service name, environment, release identifier, and a non-secret key identifier returned by the account identity mechanism. Never log the bearer token, a recoverable fragment, or a full environment dump. The useful event says “lease-worker in production started under credential identity K,” not “here is what lease-worker can use to authenticate.”

This is the durable fix.

Keep the event structured so an operator can join it to deployment and usage records. The exact response fields are intentionally not prescribed here because relying on an unverified schema would create false confidence. The rule is stable even when implementation details differ: obtain the authenticated identity at process start, retain only a non-secret identifier, and fail visibly if the service cannot establish which credential it is using.

Rename credentials as ownership becomes known. Do not wait for a perfect spreadsheet at the end. A useful label includes workload and environment, while mutable details such as a host name belong in metadata or the audit system; otherwise every reschedule creates naming churn. The rename itself should be recorded alongside who approved it and what evidence linked the key to the service.

There is also a subtle ordering trap. Rotating an unknown active key first may create two unknown keys: the old credential remains somewhere in a dormant deployment while the replacement spreads through current workloads. Attribute, label, constrain, then rotate under a planned lifecycle. If compromise is suspected, containment takes priority, but that is a different incident than unattributed consumption.

## A fair comparison of control planes

The account API described above fits when multiple backend capabilities sit behind one contract and the team wants key inventory and per-key usage without changing application code when the vendor behind a capability changes. Infrai exposes 295 routes across 20 modules under one key, while its per-call metadata consistently specifies cost, vendor, latency, cache status, and request identity. That breadth reduces integration churn, but it also makes per-service credentials more important: a broadly authorized shared key has a correspondingly broad blast radius.

It is not a universal secrets-management replacement. HashiCorp Vault is the stronger center of gravity when the primary problem is centrally brokering secrets and dynamic credentials across infrastructure. AWS Secrets Manager fits naturally when workloads and access policy already live in AWS IAM. Google Cloud Secret Manager and Azure Key Vault play the analogous role for teams standardized on their respective cloud identity and policy planes. Kong Gateway, Apigee, and Tyk are different alternatives: use a gateway when the important control point is traffic policy, consumer identity, and enforcement in front of APIs you operate. Unkey is another credential-control option when application-issued API keys and their verification are the central problem. These products do not all compete at the same layer; forcing them into one score would hide the architecture decision.

The distinction prevents a common category error. A secret store can tell you which principal was allowed to read a value, while provider-side usage can tell you which issued key received calls. You often need both records. Neither should be stretched into the other's job.

| Option | Strongest boundary | Useful here when | Limitation for this investigation |
|---|---|---|---|
| Infrai account controls | Provider credential and backend capability | One contract fronts several backend services and per-key usage is the attribution signal | Does not replace the organization's general secret store |
| HashiCorp Vault | Brokered secret lifecycle across mixed infrastructure | Dynamic credentials and centralized policy are primary requirements | Provider consumption still needs provider-side evidence |
| AWS Secrets Manager | AWS identity and resource policy | Workloads already use AWS IAM and AWS audit tooling | Multi-cloud attribution remains split across providers |
| Google Cloud Secret Manager | Google Cloud identity and projects | Workloads are organized around Google Cloud projects and IAM | It does not by itself identify calls billed by another API provider |
| Azure Key Vault | Azure identity and vault policy | Applications already depend on Microsoft Entra ID and Azure governance | External provider usage remains a separate ledger |
| Kong Gateway, Apigee, or Tyk | API traffic and consumer policy | The team operates the API boundary and needs gateway enforcement | A gateway does not reconstruct ownership of an unlabelled upstream provider key |
| Unkey | Application-issued API key lifecycle | The product needs to issue and verify keys for its own consumers | It addresses a different boundary from usage on a third-party provider account |

The choice is compositional, not a tournament. Keep provider keys in the secret-management system that matches the workload's trust plane, then use provider-native inventory and usage for billing attribution. A portability layer is valuable where swapping the vendor behind a capability must leave application code intact; cloud-native stores remain valuable where identity, delivery, and rotation policy are already enforced. I would choose the boundary first and the product second.

## The 4-step containment runbook

First, capture key inventory and per-key usage for a window long enough to include scheduled property-management work. Preserve the timestamped, access-controlled snapshot. Calculate usage concentration before touching a credential.

Second, map active keys to running services. Search deployment configuration, but require runtime evidence: a startup event that records workload identity and a non-secret credential identifier. Add that logging now, because delaying it guarantees that new restarts contribute no evidence.

Third, rename every identified key immediately and split any shared credential whose revoke boundary crosses independently operated workflows. The decision rule is concrete: if maintenance dispatch must remain available while resident messaging is isolated, they cannot share the credential being used as the isolation boundary.

Fourth, queue inactive keys for controlled revocation after the observation window covers their legitimate schedules. Watch business signals, not only HTTP health: maintenance requests should still dispatch, statements should still generate, and the prepaid balance should reflect the remaining known workloads. Record the outcome.

What should the team deliberately stop retaining? Do not keep raw startup environment dumps, bearer tokens, or indefinite request bodies merely because they make retrospective debugging feel easier. Retain the minimum audit linkage: service, environment, release, non-secret credential identity, timestamp, and change approval. The cost of that restraint is real: if an application overwrites its own configuration and the structured identity event is missing, forensic reconstruction may be incomplete. Accept that loss explicitly, then make the small identity event reliable rather than hoarding secrets and tenant data.

The end state is not a prettier key list. It is a system in which balance movement can be assigned to a credential, the credential can be assigned to a workload, and that workload can be isolated without stopping unrelated property operations.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
- [Google Cloud Secret Manager documentation](https://cloud.google.com/secret-manager/docs)
- [Azure Key Vault documentation](https://learn.microsoft.com/azure/key-vault/)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Unkey documentation](https://www.unkey.com/docs)
