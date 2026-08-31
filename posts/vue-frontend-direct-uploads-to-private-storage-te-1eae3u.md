# Vue Frontend Direct Uploads to Private Storage: Tenant-Bound Documents

Short answer: for an e-commerce application that must retain signed documents until an explicit deletion deadline, keep tenant authority and retention state in the Node backend and database, send the bytes from Vue directly to private object storage, and treat Axios progress as transport feedback rather than proof of attachment or compliance. The trade-off is deliberate: fewer application-server bytes, more state to reconcile.

The upload widget is the easy part. The difficult part is ensuring that a document cannot cross a tenant boundary, that a client filename cannot define a storage namespace, and that a green progress bar cannot be mistaken for a durable business record. A system that gets those distinctions right can change storage providers later; a system that gets them wrong merely moves its ambiguity into a bucket.

The bar is not the receipt.

## Start with the deletion deadline and the tenant boundary

An uploaded contract belongs to an order, a merchant, a buyer, and a retention policy. It is not just a blob. Record the explicit deletion deadline before issuing upload authority, and derive the object key on the server from authenticated tenant context plus a server-generated document identifier. The browser may provide a display filename; it must not choose the tenant prefix, owner, or expiry.

The database record should include tenant ID, object key, purpose, upload state, creation time, and deletion deadline. Object storage persists bytes; it should not be the only place where policy lives. That record gives the read path something stable to authorize and gives a deletion worker a bounded query instead of a filename search.

Tenant prefixes help, but they are not authorization. A callback that accepts a key supplied by another tenant, or a deletion query without a tenant predicate, can defeat private storage even when the bucket itself is configured correctly. Test those boundaries as deliberately as the happy path.

## How should Vue, Axios, presigned URLs, and a Node backend divide upload work?

The Node backend authenticates the caller, creates an upload intent, assigns the object key and deadline, and returns a narrowly scoped presigned capability. Vue sends the file directly to private object storage. Axios can report loaded bytes and a percentage when the browser exposes a computable total; when it does not, the interface should show an indeterminate state rather than invent precision.

The final attachment request should contain an intent ID, not an instruction to trust an arbitrary key. Node checks that the intent belongs to the authenticated tenant, that the key matches the server-created value, and that the storage operation completed according to the provider interface. Only then does it create or update the signed-document record. A browser can display 100% while that request is still pending. Don't call that complete.

Here is a storage-neutral policy model. The provider-specific presign and metadata calls belong behind the named interfaces, while tenant checks and state transitions remain visible to tests.

```python
from dataclasses import dataclass
from datetime import datetime
from uuid import UUID


@dataclass(frozen=True)
class UploadIntent:
    intent_id: UUID
    tenant_id: str
    object_key: str
    delete_at: datetime
    state: str


def create_signed_document_intent(
    tenant_id: str, order_id: str, delete_at: datetime
) -> tuple[UploadIntent, str]:
    if not tenant_id:
        raise ValueError("authenticated tenant is required")
    if delete_at <= now_utc():
        raise ValueError("deletion deadline must be in the future")

    intent = save_intent(
        tenant_id=tenant_id,
        object_key=server_generated_key(tenant_id, order_id),
        delete_at=delete_at,
        state="authorized",
    )
    capability = issue_presigned_upload(intent.object_key)
    return intent, capability


def attach_after_upload(tenant_id: str, intent_id: UUID, object_key: str) -> None:
    intent = load_intent(intent_id)
    if intent.tenant_id != tenant_id:
        raise PermissionError("intent is outside the tenant")
    if intent.object_key != object_key or intent.state != "authorized":
        raise ValueError("upload intent does not match")
    verify_object_metadata(object_key)
    mark_attached(intent_id)
```

The function names are policy seams, not claims about a particular SDK. In production, `verify_object_metadata` should use the documented metadata or head operation for the selected storage service, and `mark_attached` should use the database's normal concurrency controls. Make the attachment operation idempotent: a retry after a lost response must not create a second document record.

## What does an Axios progress bar prove about private object storage?

Only what the client event means. It can describe bytes observed by the browser; it cannot prove that the object is attached to the intended order, that another tenant cannot read it, or that its deletion clock is recorded. This distinction should be visible in the UI: “transferring,” “confirming,” and “attached” are different states.

Failure modes are easier to reason about when they have names:

- The user closes the tab after authorization but before the transfer.
- The transfer completes, but the attachment request never reaches Node.
- A timeout causes the browser to submit the same intent twice.
- A stale or cross-tenant intent is presented with an object key that exists.
- A deadline changes while a deletion job is already queued.
- The database says deleted while the storage operation has not been recorded.

The most dangerous result is not a red progress bar. It is an orphaned object that looks successful, has no attached business record, and later cannot be classified by the retention worker. Give intents and documents stable IDs and timestamps. Log tenant ID, intent ID, state transitions, and object key where useful, but never put capability URLs or document contents in ordinary application logs. I've kept those as separate audit fields in designs because a URL that grants access is not harmless diagnostic text.

Consider the ordinary-looking timeout. The browser uploads the final byte, Axios emits its last progress event, and the confirmation request reaches the backend just as the client loses connectivity. On retry, a naive endpoint inserts a second row; on cleanup, a naive worker sees the first object without an attached row and deletes it; on support review, the progress screenshot appears to prove that everything worked. A durable intent resolves that ambiguity: the retry uses the same ID, the state transition is idempotent, and reconciliation can distinguish “authorized but never transferred” from “transferred but awaiting attachment.” That sequence is why the database record, storage metadata, and audit event should be designed together instead of added one at a time after the upload UI ships.

Deletion needs the same discipline as upload. A worker selects records whose deadline has passed, rechecks state and tenant scope, performs the storage deletion, and records the result. If the storage service has asynchronous cleanup or multipart-upload rules, the worker needs bounded retries and reconciliation. A successful database update alone is not proof that the bytes are gone.

## Which storage interface fits the retention contract?

The useful comparison comes after the policy is explicit. A native provider interface exposes more provider-specific controls, while a storage-neutral adapter keeps business code narrower. Neither choice fixes a weak intent model.

| Boundary | Benefit | Cost or unsuitable case |
| --- | --- | --- |
| Native storage interface | Full access to the selected service's metadata, lifecycle, and retention controls | A poor fit when several backends must share one limited policy surface and the team cannot carry provider-specific behavior |
| Storage-neutral adapter | Stable application policy across backends and simpler test doubles | Not suitable when legal or operational requirements depend on versioning, retention lock, replication, or another control the adapter does not expose |
| Browser direct transfer | The application process does not relay document bytes | Requires a tested CORS and capability contract for the actual browser origin and headers |
| Server-controlled object key | Prevents client filenames from defining tenant namespace | Moving or renaming an object becomes an application workflow |
| Database-owned deadline | Makes retention queryable, auditable, and testable | Requires a worker and reconciliation path |

Stick with a native interface when a specific storage control is part of the contract. Use an adapter when portability is the real requirement and its narrower feature set is acceptable. The comparison axis is isolation and evidence, not a demo's transfer speed.

## Roll out the browser-to-storage path in small slices

Start with one tenant-scoped intent path and a test corpus containing small files, large files, duplicate submissions, expired deadlines, and interrupted browser sessions. Assert that every key is server-derived and that a cross-tenant attachment is rejected even when its object exists. Test reads separately from writes; a successful upload does not establish permission to read.

Before production, rehearse deletion against disposable objects and compare three records: the database deadline, the worker's decision, and the storage operation result. Monitor authorization-to-attachment latency, orphaned intents, expired records awaiting deletion, and reconciliation volume by tenant. Your mileage may vary across browsers and storage services, so define operational measures around attached intents and deletion evidence rather than treating a single progress event as universal.

The ownership line should remain boring and explicit: Vue owns interaction and honest progress; Axios owns the browser request; Node owns identity, naming, intent state, and attachment authorization; private object storage owns bytes under the capability; the database and worker own the retention decision. That is the design that keeps a signed document in the right tenant and out of storage after its deadline.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://cloud.google.com/storage/docs
