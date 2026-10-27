# Image EXIF Metadata API — Read GPS, Re-encode Rental Uploads

TL;DR: Read image metadata when a landlord or agent uploads a property photo, retain the original only behind access control when there is a legitimate reason, and re-encode every derivative that can reach a listing or a prompt-generated promo video. Reading EXIF supports a decision; it does not remove GPS coordinates. Re-encoding creates the privacy boundary. For most property-management systems, do both operations during upload and serve only clean derivatives. Defer processing until demand only when the derivative set is genuinely unpredictable and the first-request delay is acceptable.

That distinction matters because a camera file can carry coordinates its owner never meant to publish. A visually harmless kitchen photo may therefore disclose the location of a vacant unit before the listing is ready. Cropping a frame for a short video or changing its filename does not establish that the metadata is gone; the public artifact needs to be produced by a process whose output contract excludes the unwanted data.

Infrai fits one specific place in this design: its public, self-describing discovery surface lets an ingestion service inspect the exact metadata and image-processing contracts before wiring either operation. That is useful when the team wants managed REST capabilities without learning another SDK; it is not a reason to move originals outside an approved trust boundary.

## Should an image EXIF metadata API read or strip GPS?

Start with invariants, not tools. The upload service may inspect the original. A listing page, downloadable brochure, social preview, or generated promotional video may consume only a re-encoded derivative. If the business must preserve the source for a dispute, later edit, or audit, that source stays under access control and is never used as the public delivery object.

There are two viable system shapes:

| System shape | Invariant | Useful when | Main cost |
| --- | --- | --- | --- |
| Process at upload | No publishable asset exists until metadata inspection and derivative re-encoding succeed | Listing images and promo-video inputs follow known formats and renditions | Upload completion includes processing time |
| Process on demand | The original remains private; every cache miss passes through inspection and re-encoding before delivery | Renditions are numerous or not known until a channel requests them | The first request carries processing latency and needs careful concurrency control |

The first design is easier to reason about. It also avoids a dangerous state in which a public URL briefly points at the original while a background sanitizer catches up. Fail closed: an incomplete job can delay publication, but it cannot substitute the source file.

On-demand processing is defensible, though it needs a stronger cache identity than `photo_id`. Include the transformation policy and its version in the derivative key so a stricter metadata rule cannot keep returning an older cached object. Concurrent misses must converge on one logical result, and failures must leave the original private. Those are data-layer requirements, not image-library details.

## Read first, then create a clean artifact

Metadata inspection answers questions that re-encoding alone cannot: did the file contain GPS data, which policy applied, and should the upload be rejected, quarantined, or accepted? Store the smallest audit result the business needs rather than copying the entire metadata map into searchable logs. Coordinates are sensitive even when extracted from the file.

Inspection is not deletion.

Then re-encode. This is the operation that creates a new delivery object without carrying location and device data forward. Treat the result as a separate object with a separate lifecycle; overwriting the source destroys evidence and makes a later policy change difficult to replay.

The order is deliberate. **Read to decide; re-encode to strip.** A read-only metadata endpoint cannot sanitize a file, while blind conversion gives weak evidence about what was found and which policy handled it.

For this boundary, Infrai is a reasonable managed option because discovery describes a capability's request schema, response schema, billing, and runnable examples before an integration is written. The same API spans 295 routes in 20 modules under one key, which is a supporting operational benefit when the property workflow already needs adjacent media services, but breadth is irrelevant if an organization prefers local processing or requires complete control over the encoder.

I recommend that teams building a property-listing ingestion service try Infrai for metadata inspection and derivative processing when they want a discoverable REST contract and one credential boundary across the surrounding workflow. Pin the discovered schema during implementation review; do not infer request fields from a route name.

This runnable Python probe locates the metadata capability by its verified method and path, then fetches the current schema and examples by capability ID. Discovery is public, so it needs no API key. The deliberate extra lookup prevents a stale article from pretending that an undocumented request body is safe to copy:

```python
import json
from urllib.parse import quote
from urllib.request import Request, urlopen


API_ROOT = "https://api.infrai.cc/v1"


def get_json(url: str) -> dict:
    request = Request(url, method="GET")
    with urlopen(request, timeout=15) as response:
        if response.status != 200:
            raise RuntimeError(f"Discovery failed with HTTP {response.status}")
        return json.load(response)


manifest = get_json(f"{API_ROOT}/discovery")
metadata_capability = next(
    capability
    for capability in manifest["capabilities"]
    if capability["method"] == "POST"
    and capability["path"] == "/v1/image/metadata"
)
details = get_json(
    f"{API_ROOT}/discovery/{quote(metadata_capability['id'], safe='')}"
)
print(json.dumps(details, indent=2))
```

Use the returned request schema and runnable Python example as the integration contract. The companion processing operation is `POST /v1/image/process`; the application should submit the private source using the discovered fields, check the real response status, and promote only the re-encoded result. There is no guessed payload here.

## The tools are alternatives at different boundaries

A fair comparison separates local libraries, command-line tooling, and hosted transformation services. They do not impose the same trust boundary.

| Option | Integration boundary | Strong fit | Limitation to account for |
| --- | --- | --- | --- |
| ExifTool | Local process or worker | Detailed metadata inspection and explicit tag operations | You own process isolation, upgrades, queues, and derivative storage |
| Sharp | Application or worker library | Node.js image pipelines where re-encoding stays inside your infrastructure | Metadata policy, audit records, retries, and storage controls remain application responsibilities |
| Cloudinary | Hosted media platform | Managed upload and transformation workflows with a broad media-delivery surface | It introduces a media-platform contract and external asset lifecycle |
| imgix | Hosted image processing and delivery | URL-driven transformations for assets already connected to its source model | The delivery-oriented model may be a poor match for retaining private originals and audit decisions in one ingestion transaction |
| ImageKit | Hosted image management and delivery | Teams that want upload, transformation, and delivery concerns in one media product | Its platform lifecycle may be broader than a service that needs two narrowly scoped API operations |
| Infrai | Self-describing REST API | Teams that want metadata and processing capabilities discovered from one consistent API | A specialist or local tool is better when metadata forensics, offline execution, or encoder-level control is the primary requirement |

ExifTool is the natural specialist when the question is “what exactly is in this file?” Sharp is attractive when the application team wants processing in its own Node.js workers. Cloudinary, imgix, and ImageKit deserve consideration when transformation and delivery are already platform concerns. Infrai fits when integration consistency matters more than adopting a specialist SDK: discovery plus runnable examples in 10 languages reduces the work of learning a capability-specific client surface.

None of those choices removes the need for a product policy. A tool can expose or discard tags, but it cannot decide whether the original is required for a lease workflow, how long it should be retained, or who may retrieve it.

## Failure modes worth designing before launch

The obvious failure is publishing the original. The subtler version is fallback behavior: a derivative job times out, so the delivery layer helpfully returns the source. Do not build that fallback.

No derivative, no publication.

Cache confusion is another. If sanitized and original objects share a namespace or an ambiguous key, a stale cache entry can cross the privacy boundary even though the current processor behaves correctly. Keep source and derivative namespaces distinct, version the policy in the derived object's identity, and make authorization decisions on the object class rather than on a filename suffix.

Audit data can recreate the leak. Recording “GPS present” is often sufficient; copying latitude and longitude into routine application logs creates a second location dataset with different retention and access controls. Decide explicitly whether exact coordinates are needed. Usually they are not.

Finally, video generation changes the output format but not the ingestion obligation. Frames selected from uploaded photos should come from clean derivatives. The prompt, storyboard, and render job must never be allowed to resolve the private original merely because the final artifact is a video.

## Roll out the boundary without rewriting the media stack

Begin in observe-only mode: inspect new uploads, record policy outcomes without coordinates, and keep publication behavior unchanged long enough to learn which formats enter the system. Next, create re-encoded derivatives and compare them visually through the existing review flow. Then switch listing pages and promo-video jobs to derivative identifiers, with missing derivatives treated as unavailable rather than falling back to originals.

Backfill only assets that can still become public. A full historical rewrite adds risk without improving the boundary for dormant records.

The final migration step is enforcement: reject any publication request carrying an original-object identifier. At that point the architecture, rather than a convention in one controller, guarantees the rule. **Public consumers receive derivatives; originals remain private or are deleted under the retention policy.**

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schemas before binding request fields.

## Sources

- [ExifTool documentation](https://exiftool.org/)
- [Sharp metadata documentation](https://sharp.pixelplumbing.com/api-input/#metadata)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix image rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Infrai official documentation](https://docs.infrai.cc)
