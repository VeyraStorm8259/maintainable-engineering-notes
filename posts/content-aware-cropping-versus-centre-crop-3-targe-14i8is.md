# Content-Aware Cropping Versus Centre Crop: 3 Target Aspect Decisions for Promo Videos

Short answer: content-aware cropping chooses *where* to place a fixed-aspect window using evidence about the subject; center cropping puts that window at the geometric center. Neither method can preserve content that will not fit inside the target aspect ratio. For a marketplace generating short promo videos from prompts, the useful decision is often upstream: freeze an approved thumbnail at upload or render it on demand, then record exactly which source and crop decision produced it. A visually clever crop is little consolation if a product label disappears between approval and publication.

## What does content-aware cropping actually do versus centre crop?

Consider a 1600 by 900 video frame destined for a 1:1 listing thumbnail. A square crop without resizing takes 900 by 900 pixels, leaving 700 horizontal pixels outside its window. Center crop starts at x = 350. Content-aware crop still uses a 900 by 900 window, but may shift its left edge anywhere from x = 0 to x = 700 after evaluating the image. Those dimensions follow from the geometry, not from a promise about any particular detection model.

What counts as content depends on the implementation: a face, a detected object, a saliency map, or a manually supplied focal point can each lead to a different window. A high-scoring region is not necessarily the right region for a seller's listing. A frame with a person's face on the left and the item being sold on the right exposes the mismatch immediately; a face-prioritizing crop may preserve the person and cut away the item. Center crop has a different failure: it has no notion of either subject. Its virtue is determinism. Neither score nor symmetry establishes which element the buyer came to inspect, so the definition of the subject has to be part of the placement's acceptance criteria, not an undocumented assumption inside the image worker. If a listing uses the identical generated frame for a square tile and a vertical promo preview, inspect both outputs: an acceptable square can conceal a disastrous vertical crop.

The target ratio also matters independently of the algorithm. A 9:16 teaser drawn from a 16:9 source discards substantially more horizontal field than a square thumbnail; moving the window cannot restore what lies beyond its edges. If preserving both subjects is mandatory, the options are a different source frame, a composed canvas with padding, or separate assets for the two placements.

Do not call padding a crop. It changes the output composition.

## When should the decision become durable?

For prompt-generated marketplace promos, generation, moderation, seller review, and publication need to agree on which frame is being shown. If a thumbnail is approved with one focal point but a later request runs a revised detector, the same listing can present a different product. That is a consistency failure even if both renders are valid images. Store a source identifier, source revision, target dimensions, crop rectangle, orientation policy, and algorithm or annotation revision with the approved derivative. Keep the original available for new placements, subject to the applicable retention policy.

Processing at upload makes that approval boundary straightforward: generate each known placement, inspect it, and publish an immutable derivative reference only after checks pass. Its limitation is extra storage and processing for frames that may never be displayed, so it is not suitable when placements change rapidly and most derivatives go unused. On-demand processing accommodates new aspect ratios without regenerating everything, but it needs a stable transformation key, cache invalidation rules, and a policy for what happens when detection changes. Its limitation is that a request-time fallback can silently change the image under review unless approvals bind to the exact output. A cached image without its decision metadata is hard to audit.

Here is the narrow calculation that both paths need. It returns a bounded crop rectangle from a normalized focal point; it does not pretend to infer what the focal point should be.

```python
def crop_box(width: int, height: int, target_w: int, target_h: int,
             focal_x: float = 0.5, focal_y: float = 0.5) -> tuple[int, int, int, int]:
    if min(width, height, target_w, target_h) <= 0:
        raise ValueError("dimensions must be positive")
    if not (0 <= focal_x <= 1 and 0 <= focal_y <= 1):
        raise ValueError("focal coordinates must be normalized")

    if width * target_h >= height * target_w:
        crop_h = height
        crop_w = min(width, round(height * target_w / target_h))
    else:
        crop_w = width
        crop_h = min(height, round(width * target_h / target_w))

    left = max(0, min(width - crop_w, round(focal_x * width - crop_w / 2)))
    top = max(0, min(height - crop_h, round(focal_y * height - crop_h / 2)))
    return left, top, left + crop_w, top + crop_h
```

Rounding can make the pixel ratio approximate rather than exact; resize the extracted rectangle to the required output dimensions, and test the actual encoder output. Decode the source consistently before interpreting coordinates, since image orientation metadata can change which direction a pixel coordinate points. Format support also varies by browser and image type, as the MDN image format guide documents; choose a delivery format through compatibility tests instead of treating the crop rectangle as the whole pipeline.

## Which failure should determine the policy?

The comparison belongs after the invariant: the published asset should remain tied to the source and decision that passed review. Neither placement strategy supplies that guarantee by itself.

| Strategy | Useful boundary | Failure to test |
| --- | --- | --- |
| Center window | Consistent framing when the subject is reliably centered | Off-center merchandise is clipped without warning |
| Detected focal window | Variable subject placement with a trustworthy detection target | The detector favors a face, text, or background over the sale item |
| Human-specified focal window | Reviewed hero placements and ambiguous multi-subject frames | An annotation becomes stale when the source frame changes |
| Padded composition | Both edges must remain visible at a narrow target ratio | Small subjects and letterboxing may violate placement requirements |

Test with frames that contain multiple subjects, text near the edge, empty backgrounds, and products partly outside the source. For each target ratio, inspect the final encoded image at the display size, not just the large crop preview. Record clipping and wrong-subject outcomes separately: an image may keep the detected object intact yet still be commercially wrong. There is no meaningful universal accuracy number here without a labeled set of representative listings and a definition of the intended subject.

Failed decoding, unsupported inputs, missing focal metadata, and timeouts need explicit outcomes. A center fallback can keep a thumbnail available, but it must be marked as a fallback and should not silently inherit approval from a different crop. Monitor the rate of fallback, manual overrides, crop changes by algorithm revision, and review rejections by placement. Storage consumption and generation latency matter too; measure them against the marketplace's actual publication and request patterns instead of assuming that either upload-time or on-demand processing is cheaper. The counterexample worth keeping in the test set is the ambiguous frame: a crop can be perfectly valid in dimensions, pass decoding and encoding, and still show the wrong item. Count that as a failure of subject selection, rather than allowing a successful image response to mask it.

Pixels aren't approvals.

## How do you roll it out without changing approved listings?

Start with one placement and a fixed set of reviewed source frames. Generate candidate rectangles alongside the existing outputs, compare encoded thumbnails, and attach decision metadata without replacing published references. Promote a crop only after the review path can approve the exact derivative that buyers will see. New algorithm revisions should create new derivative identifiers; previously approved listings retain their earlier ones until a deliberate re-review. This also gives the team a reversible path when an apparently sensible focal rule starts cropping out the actual merchandise.

## References

- MDN, Image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
