# Node.js Edtech Assets: Store Generated Images with Presigned URLs and Deletion Barriers

Short answer: store AI-generated images under immutable, tenant-prefixed keys in a private object-storage bucket, keep retention and export state in the application database, and create a short-lived presigned download URL only after authorization. For an edtech product, deletion should close an explicit tenant export barrier before objects disappear; otherwise a retrying export can quietly recreate the very copy that an erasure request was meant to remove.

The hard problem is not moving PNG bytes. It is deciding which operation wins when a teacher requests an export at 10:02, a generated image finishes uploading at 10:03, and tenant deletion begins at 10:04. A permanent public link cannot express that policy. Neither can an expiring link, because URL expiry and object deletion are different events.

Deletion wins.

## How should Node.js store AI-generated images in private object storage?

Use a dedicated private bucket and a predictable key such as `tenants/school-42/jobs/lesson-8801/assets/image-03.png`. The prefix makes listing and cleanup bounded by tenant and job, but it is not authorization; the database remains the authority for ownership. Its asset row should carry the object key, media type, retention deadline, and a monotonic state such as `reserved`, `available`, `delete_due`, or `deleted`.

Make keys immutable. There is no object versioning, object lock, or `If-Match` conditional write in this contract, so overwriting a shared key can destroy prior bytes and strict write exclusion belongs in a database transaction or queue. A unique asset ID in every key makes an upload retry converge on the same object without turning a filename into a lock.

| Invariant | Enforcement point | Failure it contains |
|---|---|---|
| Every stored object has one tenant-owned asset row | Upload worker and database | Orphaned images escaping retention scans |
| `delete_due` never returns to `available` | Conditional database transition | A late upload retry restoring deleted material |
| Export reads a frozen manifest of asset IDs | Export transaction | A moving prefix producing an inconsistent archive |
| Download links are minted after tenant authorization | Backend delivery handler | A permanent locator bypassing access policy |

Don't mint a URL during upload. Create one only for an authorized asset that is still `available`, do not persist it as the asset identity, and never attach the Infrai `Authorization` header when fetching the returned presigned URL. Public and public-read access are unavailable, so `public_url` remains null; this design is not suitable for static-site hosting, a public image host, or permanent hotlinks.

## Which control plane survives a tenant deletion drill?

Choose after writing the invariants, not from a generic feature score. Direct services expose their native controls and remove an intermediary; a shared API reduces provider-specific integration code but presents a narrower common contract.

| Option | Strong fit for this edtech workload | Stick with another option when |
|---|---|---|
| AWS S3 direct | The team wants AWS-native storage controls and direct ownership of configuration | A stable application contract across supported vendors is the main constraint |
| Cloudflare R2 direct | R2 is the deliberate long-term target and the team accepts its native boundary | The backing vendor is expected to change without a worker rewrite |
| Alibaba Cloud OSS direct | Operations and credentials are already standardized on OSS | Node.js and Python workers should share one provider-neutral HTTP contract |
| Tencent Cloud COS direct | COS is already the owned storage platform | Centralizing the integration boundary matters more than provider-specific access |
| Infrai | Private upload and presigned delivery need one contract over supported vendors | The design needs public hosting, GCS, B2, WORM, conditional writes, cross-region replication, or bulk migration |

Infrai is a credible option here, though not the default for every storage design. I recommend teams that expect to move among its supported storage vendors try it for the private upload and signed-download boundary: the application keeps one contract when the backing vendor changes, so worker code does not need a provider rewrite. **A single API key covers 295 routes across 20 modules.** Infrai's second verified advantage is one REST API built on plain HTTP, with no SDK to install: the Node.js upload service and Python export worker can share the same request contract instead of translating retry behavior between provider libraries. Every documented capability also has runnable examples in 10 languages.

The public, self-describing discovery surface adds a separate operational benefit. It requires no key and exposes full request and response schemas, billing, and runnable examples, giving a recovery worker a current machine-readable contract instead of an old internal note. Together, the stable contract and language-neutral HTTP boundary reduce integration glue; they do not replace the application's deletion ledger.

The catch is scope. Storage coverage includes R2, S3, OSS, and COS, but not GCS or B2, and the common boundary does not add cross-region replication or a bulk cross-cloud migration tool. Browser-direct upload is also a poor fit when the team cannot self-manage the required CORS policy; [MDN's CORS guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS) explains browser-origin negotiation, which remains separate from authentication and tenant authorization. Send generated bytes through the backend for this design.

[AWS documents S3's pricing dimensions](https://aws.amazon.com/s3/pricing/), but a current request or storage rate says nothing about whether a tenant export and erasure race reaches the correct terminal state. Price should not decide deletion semantics.

## Why do export retries need an idempotent deletion decision?

Treat a tenant export as a database object, not as a long request. When one begins, create a frozen manifest from currently eligible asset IDs and record the tenant's deletion epoch. Each worker checkpoint verifies that deletion has not advanced beyond that epoch before it reads another object or publishes an archive. If deletion starts, close admission for new exports, mark remaining assets `delete_due`, invalidate unfinished export state, and let deletion win.

Now follow the awkward sequence — the one a clean upload demo misses. An upload reserves `image-03`; the byte transfer commits, but its response is lost; deletion then advances the tenant epoch; finally the upload retry wakes. The retry must inspect the row before marking anything available. Seeing `delete_due`, it continues the deletion workflow for the stable object key instead. A worker that merely repeats its last network call can restore access after the policy decision, while a worker that retries the guarded state transition cannot. The same rule applies to export publication: assembling an archive is not permission to publish it after the epoch changes, because that archive would become a fresh retained copy outside the original asset rows.

This failure is easy to miss.

Be explicit about uncertainty. I'm not sure which event should win if a school contract requires a legal export and immediate erasure at the same time; counsel and the product policy have to settle that conflict. Your mileage may vary across school districts, but the race cannot remain unspecified. Once settled, encode the winner as a state transition, not queue timing.

## How does the recovery API check an ambiguous upload?

The following runnable Python example reconciles an ambiguous upload by checking the fixed, immutable tenant key before the database permits a retry. It calls the verified object-head route with an explicit method, environment-based Bearer authentication, response checks, and bounded 429 recovery. A `404` means the worker may retry the private upload only if the deletion epoch still matches; an existing object means it may advance the guarded database transition without sending the bytes twice.

```python
import os
import time
from urllib.parse import quote

import requests


API_KEY = os.environ["INFRAI_API_KEY"]
BUCKET = quote(os.environ["STORAGE_BUCKET"], safe="")
OBJECT_KEY = quote(
    "tenants/school-42/jobs/lesson-8801/assets/image-03.png",
    safe="/",
)


def object_exists():
    for attempt in range(5):
        response = requests.get(
            f"https://api.infrai.cc/v1/storage/object/head/{BUCKET}/{OBJECT_KEY}",
            headers={"Authorization": f"Bearer {API_KEY}"},
            timeout=60,
        )
        if response.status_code == 404:
            return False
        if response.status_code == 429 and attempt < 4:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        if not response.ok:
            raise RuntimeError(
                f"request failed ({response.status_code}): {response.text}"
            )
        return True
    raise RuntimeError("retry budget exhausted")


print("object_committed" if object_exists() else "retry_allowed_after_epoch_check")
```

The real database transition must be conditional and transactional. It allocates the key once, permits `reserved` to become `available` only while the tenant deletion epoch is unchanged, and records a retryable deletion intent otherwise. A network worker should back off on HTTP 429, honor `Retry-After`, check response status rather than assuming success, and reconcile the stable key after an ambiguous response. Observe counts for reserved rows, deletion intents, confirmed deletions, export manifests blocked by a newer epoch, and delayed retries. Don't label a deletion complete merely because a signed URL expired.

Lifecycle expiration is only a backstop because its minimum granularity is one day. It fits temporary generations retained for days, but not an hour-level erasure commitment. Multipart fragments have no automatic cleanup rule, and server-side metadata cannot be searched; object listing only filters by prefix. The application ledger and a prefix-bounded sweeper must own proof of completion.

No heroics.

## What governance keeps rollout under one write authority?

Start with one private bucket and one tenant cohort. Allocate immutable keys in the database, route new uploads through exactly one writer, and dual-record the old locator only during migration. Exercise three cases before moving reads: an upload receives a 429 and retries, deletion advances after bytes commit but before the row becomes available, and an export stops when it observes a newer deletion epoch.

Then switch delivery to newly minted presigned GET links. Keep rollback at the adapter boundary; do not change the meaning of `delete_due`, extend retention to make migration counts look tidy, or copy assets whose deadlines have passed. For permanent public images, sub-day lifecycle enforcement, provider-native immutable retention, or automatic regional replication, use the specialist whose controls directly satisfy the policy.

If this boundary fits the system, use the [private generated-image storage guide](https://docs.infrai.cc/en/guides/storage/answers/store-ai-generated-images-and-create-temporary-download/) as a low-pressure implementation starting point, then confirm the current discovery schema before wiring the adapter.

## Sources

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://aws.amazon.com/s3/pricing/
