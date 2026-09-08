# Tenant Isolation Beats a Public URL: Signed Download Windows for Private Uploads

A private bucket does not hand out a public URL, and that is the entire explanation behind the image that stopped rendering: the bytes are readable only by a request carrying a valid signature, so the application mints that signature on demand and it expires on a clock you picked. The object itself is fine. In a customer support product — tenants, their agents, and their end users all pushing screenshots, invoices and log dumps into tickets — the question worth arguing about is not how long a signed download link should live, but which objects the credential that signed it was ever able to reach.

So, the decision for this system: use one bucket per tenant while the tenant count stays comfortably under the provider's bucket quota, sign every download against that tenant's prefix with a credential that cannot address any other prefix, keep the window in minutes, and allow no anonymous read anywhere in the account. Avatars in the agent console are treated exactly like ticket attachments, because the day one prefix goes public is the day someone discovers a customer's ID scan in it.

Get the isolation boundary right and the expiry window becomes a tuning knob. Get it backwards and you have a tidy 10-minute link that cheerfully serves another tenant's invoice.

## What does a private object storage bucket do to a public image URL?

Anonymous read is a policy, not a property of the bytes. A public object is one the bucket policy lets an unauthenticated GET fetch; make the bucket private and every GET has to prove who it is. A browser cannot do that on your behalf, since it has no business holding your storage credentials and an `<img>` tag cannot attach an `Authorization` header anyway.

Query-string signing exists for exactly that gap. The signature covers the method, the bucket, the key, an expiry and the credential scope, then rides in the query string so a plain `<img src>` works with no headers at all. Signature Version 4 caps a presigned GET at seven days, and when you sign with temporary session credentials the link dies with those credentials, whichever comes first. Nothing about that is a defect — it is the contract you asked for when you turned off anonymous access.

The part teams underestimate is that the URL is now a bearer capability. Anyone holding the string is the caller until the clock runs out: it lands in browser history, in `Referer` headers on any page you embed it in, in CDN and proxy access logs, and — in a support tool, of all places — in the reply an agent pastes into the ticket while trying to be helpful. A short window is containment, not decoration. It is also the only containment you get, because a signed URL cannot be revoked; once minted, your kill switches are deleting or moving the object, or rotating the signing credential and invalidating every URL it ever produced.

That is also the explanation for the avatar that renders at 09:00 and stops working on a tab left open until lunch. The image was never public, the session outlived the signature, and the correct repair is a refresh call — not a public bucket.

## Four invariants that keep tenant data separated

Four invariants are worth writing into the ADR, because each one exists to block a specific failure I can name.

No object is anonymously readable, and there is no exception for "just avatars". Prefix separation without a policy that denies public read is one console click away from publishing everything, and object keys are not secrets — they show up in logs, in support exports, and in any client that ever received them.

Every signature is produced by a code path that has already resolved the caller's tenant, and the credential doing the signing is scoped so it physically cannot address another tenant's prefix. This is the one that actually bites. The classic shape is an endpoint that authorizes the ticket, then obediently signs whatever object key the request handed it: the check and the capability drift apart, and you have cross-tenant read through a shared bucket. Scoped session credentials turn that from a logic bug into a denial at the storage layer, which is where you want your last line of defense — code review does not catch every handler, and a policy does not get tired.

Downloads are served with `Content-Disposition: attachment` and never from a hostname that shares session cookies with the application. User-uploaded files are attacker-controlled bytes; serving them inline from an origin that carries your auth cookie is how a support attachment becomes stored XSS against your own agents.

Every presign is an audit event: tenant, actor, object key, window, and the reason it was requested. You cannot log the download itself unless the bytes flow through you, so the presign record is the closest thing to an access trail you will have.

Two operational failure modes fall outside those invariants and still cost real hours. Signature validity is derived from the signing host's clock, so a machine with drifted time mints links that the storage endpoint rejects as out of skew — S3-compatible endpoints refuse requests whose timestamp is more than 15 minutes off. And an expired link returns a 403 that a browser or CDN may cache, so the follow-up refresh appears to change nothing until the cached error ages out; set explicit cache headers on error responses and you save your on-call engineer a confusing half hour.

## Three delivery contracts, side by side

The choice is really about who is allowed to read, and everything else follows.

| Delivery contract | Who can read the bytes | Where the tenant boundary lives | Revocation | Cache behavior | Honest fit |
|---|---|---|---|---|---|
| Anonymous object with a stable URL | Anyone who has or guesses the key | Nowhere; only obscurity | Delete or move the object | Ideal, one cache key forever | Assets that are genuinely public |
| Per-request signed URL | Anyone holding the URL until it expires | In the signing credential's scope | Wait out the window, rotate keys, or delete | Poor; the signature is part of the URL | Private uploads read by an authenticated UI |
| Bytes proxied through the application | Only an authenticated session | In your handler, on every request | Immediate | You own the headers entirely | Strict audit, DLP, or watermarking needs |

Signed cookies scoped to a path sit between rows two and three when a CDN is already in the path, and they solve the cache-fragmentation problem, at the cost of a boundary defined by URL path rather than by credential scope.

## The critical path, and how to test it

The signing function is the whole security model in about twenty lines, which is a reason to keep it in one place and test it directly.

```python
import os
from datetime import timedelta

import boto3
from botocore.config import Config

DOWNLOAD_TTL = timedelta(minutes=10)

storage = boto3.client(
    "s3",
    endpoint_url=os.environ["OBJECT_STORE_ENDPOINT"],  # any S3-compatible endpoint
    config=Config(signature_version="s3v4", retries={"max_attempts": 3}),
)


class CrossTenantRead(Exception):
    """Raised before signing, never after."""


def attachment_key(attachment) -> str:
    return f"tickets/{attachment.ticket_id}/{attachment.id}/{attachment.filename}"


def signed_download(session, attachment, ttl: timedelta = DOWNLOAD_TTL) -> dict:
    if session.tenant_id != attachment.tenant_id:
        raise CrossTenantRead(f"actor {session.actor_id} is outside tenant {attachment.tenant_id}")

    expires_in = int(ttl.total_seconds())
    url = storage.generate_presigned_url(
        "get_object",
        Params={
            "Bucket": bucket_for(attachment.tenant_id),
            "Key": attachment_key(attachment),
            "ResponseContentDisposition": f'attachment; filename="{attachment.filename}"',
            "ResponseContentType": attachment.content_type,
        },
        ExpiresIn=expires_in,
    )
    audit.record(
        "attachment.presigned",
        tenant=attachment.tenant_id,
        actor=session.actor_id,
        key=attachment_key(attachment),
        expires_in=expires_in,
    )
    return {"url": url, "expires_in": expires_in}
```

Two tests keep it honest, and neither needs a network. Freeze the clock, mint a URL, advance past the window and assert the endpoint rejects it — that turns "the link expires" from a claim into a property. Then hand a tenant A session an attachment record belonging to tenant B and assert `CrossTenantRead`, so the day someone refactors the handler the build tells them.

On the client, refresh before the window closes rather than after: a page that keeps images alive by re-presigning at roughly 70% of the TTL never shows a user a broken thumbnail. Watch the 403 rate at the storage edge as a product metric, not an infrastructure one, and alert on cross-tenant denials at any nonzero rate, since a real one means either an attack or a routing bug you shipped.

Direct browser uploads need CORS configured on the bucket, which is a separate axis of per-tenant configuration and one of the quieter costs of bucket-per-tenant. Plain `<img>` rendering of a signed URL does not need CORS at all — a distinction that has burned more than one afternoon.

## The design I rejected, and where it belongs instead

I rejected the public bucket behind a CDN with unguessable keys, and the trade-off is not close for this workload: support attachments are other people's private documents, obscurity is not an access control, and one policy change publishes the whole prefix at once.

It wins in a different job, though. Help-center screenshots, brand assets, marketing images and published documentation want a stable URL forever, a cache hit ratio near one, and previews rendered by email clients and social crawlers — none of which can refresh a signature or hold a session. Signed URLs are simply not suitable there, and a per-tenant private bucket is the wrong tool for a logo. Keep the two in separate buckets so the contract is enforced by configuration rather than by everyone remembering which prefix is which.

The bucket-per-tenant decision is the part I would revisit first at scale. Every provider caps buckets per account, and per-bucket lifecycle rules, CORS blocks and metric series multiply with the tenant count, so somewhere past a few hundred tenants the operational drag outgrows the isolation benefit; at that point a shared bucket with a per-tenant prefix and short-lived scoped credentials gives you the same denial at the storage layer with far less configuration. I don't have a defensible universal number for where that line sits — it depends on how much per-tenant configuration you carry and what your compliance story requires — and the honest answer is to check the current bucket quota and your own config sprawl before committing either way.

Ten minutes is a starting point for the window, not a law. Shorter windows shrink the blast radius of a leaked URL and raise your presign volume; longer ones do the reverse.

## Further reading

- MDN, Cross-Origin Resource Sharing (CORS): https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- MDN, Content-Disposition header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- MDN, Referrer-Policy header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Referrer-Policy
- AWS, Sharing objects with presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- AWS, Authenticating requests using query parameters (SigV4): https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.html
- AWS, Bucket restrictions and limitations: https://docs.aws.amazon.com/AmazonS3/latest/userguide/BucketRestrictions.html
- Cloudflare R2 documentation: https://developers.cloudflare.com/r2/
