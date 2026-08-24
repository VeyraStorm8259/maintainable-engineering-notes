# Choosing Object Storage and Database Metadata for Tenant-Isolated PDFs, DOCX, and Invoices

The operational constraint is authorization, not the shape of a folder. For a multi-tenant SaaS, the best storage pattern for user documents such as PDF invoices and DOCX files is one that keeps that decision outside the bucket: a private object store can hold bytes, but it can't be the system that decides which tenant may read them.

Short answer: use private object storage for document bytes, an immutable opaque key per object, and a database row as the authoritative tenant and document index. Put PDF, DOCX, and invoice metadata in that row; issue short-lived upload and download grants only after checking the tenant boundary.

## Which invariants should a multi-tenant SaaS keep for private document storage?

Treat these as an architecture decision record rather than a naming convention:

- Every document belongs to exactly one tenant in the database.
- The object key contains an internal tenant identifier and document identifier, not an email, invoice number, or original filename.
- A document is not readable merely because an object exists. Authorization checks the database row first.
- Upload, processing, and deletion have explicit states, so a half-finished transfer cannot look like a usable document.
- The key is immutable. A UI folder rename changes metadata, not thousands of objects.

A representative key is `tenants/<opaque-tenant-id>/documents/<opaque-document-id>/source`. The prefix helps operations group data; it is not an access-control mechanism. Searchable fields such as display name, media type, byte count, invoice period, retention class, and timestamps stay in the database. That separation avoids making object listings answer business questions they were never designed to answer, and it leaves room for a second object such as a generated PDF preview without changing the document's business identity or moving the original bytes. A folder shown in the product can therefore be renamed in one transaction even while background workers are copying, scanning, or indexing related objects.

The database lookup is intentional overhead. It is where the application already evaluates membership, role, legal hold, and deletion state, so putting the policy there gives every access path one place to test. A guessed document ID must still fail because the query includes the caller's tenant ID.

That is the boundary.

## How should the upload and download critical path work?

Create a pending row after authorization, return a short-lived upload target, verify the completed object, and only then transition the row to active. A retry should address the same document ID rather than silently create another business record. The application can expose its own routes while keeping provider-specific details behind the server boundary. In practice, this means a browser can be disconnected after sending the final chunk, a worker can retry verification, and a user can refresh the page without turning one invoice into two rows; the state machine resolves those events against one stable identifier, records the observed size and media type, and makes serving contingent on the active transition.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Document:
    document_id: str
    tenant_id: str
    object_key: str
    state: str
    media_type: str
    byte_count: int


class DocumentStore(Protocol):
    def get_for_tenant(self, document_id: str, tenant_id: str) -> Document | None: ...
    def mark_active(self, document_id: str, byte_count: int, media_type: str) -> None: ...


def begin_upload(store: DocumentStore, tenant_id: str, document_id: str,
                 media_type: str, byte_count: int) -> str:
    document = store.get_for_tenant(document_id, tenant_id)
    if document is None or document.state != "pending":
        raise PermissionError("document is not available for this tenant")
    # The server chooses the key; the client never supplies a tenant prefix.
    return document.object_key


def finalize_upload(store: DocumentStore, tenant_id: str, document_id: str,
                    observed_size: int, observed_type: str) -> None:
    document = store.get_for_tenant(document_id, tenant_id)
    if document is None or document.state != "pending":
        raise PermissionError("document is not available for this tenant")
    if observed_size <= 0 or observed_type not in {"application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document"}:
        raise ValueError("object metadata does not satisfy the document policy")
    store.mark_active(document_id, observed_size, observed_type)
```

For downloads, load the row by both document ID and tenant ID, check the caller's permission, then return a time-limited response or stream the bytes through the application. Cache policy is part of that decision: `Cache-Control: no-store` asks caches not to store a response, while `private` allows a private cache and prevents shared-cache storage. The right directive depends on sensitivity and whether a browser cache is acceptable; MDN documents this distinction.

Upload progress is a client concern, not an authorization shortcut. The browser can report transfer progress through `XMLHttpRequest` upload events, while the server remains responsible for state transitions and metadata validation. A progress bar reaching 100 percent is not proof that the object is safe to serve.

## What failure boundaries and trade-offs should the design document?

| Pattern | Suitable when | Boundary to accept |
| --- | --- | --- |
| One private bucket with opaque tenant prefixes and database metadata | Most small and medium multi-tenant services | Application policy must be applied consistently |
| Bucket or account per tenant | Contracts require dedicated controls or incompatible residency rules | Provisioning, monitoring, and policy changes multiply |
| Binary data in the relational database | Small payloads must share one transaction | Backups and database workload grow with file volume |
| Public objects with unguessable URLs | Public assets, never private records | URL secrecy is not authorization |

The catch is shared infrastructure. This pattern is unsuitable when a tenant needs administrator-owned bucket policy, hard physical separation, or a retention rule incompatible with other tenants. Use separate buckets or accounts for those tenants, but keep the same database authorization model and opaque keys so the application contract does not change.

Direct downloads reduce application bandwidth, but they move expiry, cache headers, and access logging into a signed-grant flow. Proxying through the application centralizes those checks and can simplify audits, at the cost of connections and throughput. Pick per document class after measuring transfer volume and compliance requirements.

## How do testing and operations prevent tenant leaks?

Make the cross-tenant case a first-class test: a valid user from tenant A must not read, finalize, rename, or delete tenant B's document even when the document ID is known. Add tests for expired grants, duplicate finalization, interrupted uploads, zero-byte files, incorrect media types, deletion during background processing, and filename encoding. Unit tests can cover policy; an integration test should cross the database-to-object-store boundary.

Reconciliation is part of the storage design. Periodically compare pending rows with object existence, age abandoned uploads, and report active rows whose expected objects are absent. Also find objects without database records; hold them for review before deletion because delayed finalization and retries can create a legitimate race.

Trace one document ID through planning, transfer, finalization, scanning, preview generation, and download. Log tenant and document identifiers as structured fields, while excluding original filenames and access URLs. Useful signals include pending age, finalize latency, validation failures, orphan candidates, authorization denials, and bytes by tenant.

A test batch once spent 47 minutes in a confusing state because a callback payload lacked the `size` field someone had assumed existed. The lesson is unglamorous: validate external payloads at the boundary and obtain authoritative object attributes before activation. It failed loudly. That's useful. Your mileage may vary with a particular storage service, but the invariant does not change.

The design stays explainable: private bytes, authoritative metadata, explicit states, expiring access, and a tenant check at every application boundary. Keep it boring.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
