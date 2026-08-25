# duvro

An iOS app that turns a camera roll into a ready-to-post Instagram carousel: it picks the photos, applies a coherent aesthetic, orders the post, and writes the caption.

Built solo. This repo is a writeup, not the source. The app and server live in private repos.

---

## The problem

Posting a photo dump takes about twenty minutes: scrolling a few hundred photos, picking seven, editing each one so they look like a set, ordering them, writing a caption. Most of that time is spent on decisions people don't enjoy making.

duvro compresses it to about ninety seconds without asking the user to give up control of what gets posted.

---

## Architecture: a two-stage funnel

The first design ran everything on-device. It didn't work, for a reason that turned out to be structural.

Apple's Vision framework can tell you a photo is sharp, well-exposed, and contains three faces. It cannot tell you the photo is a cooking class at a kitchen store, or that the person in it looks good, or that two photos are the same moment. Those judgments are what actually decide whether something is worth posting.

So the pipeline splits by what each stage is good at:

**Stage 1, on-device (Vision, ~2.5s per 100 photos, free)**
Reject junk. Classify into people / place / food / detail. Cluster near-duplicates. Shortlist roughly 30 candidates.

**Stage 2, one API call (Claude Sonnet, ~50s)**
Receive 30 thumbnails plus eight metrics each. Select the post, order it, identify runner-ups, write ranked captions.

Stage 1 nominates. Stage 2 selects. The split keeps the expensive call to one request per session (~$0.02) instead of sending 200 photos, and it puts the judgment where judgment exists.

**What that looks like in practice.** In one test, Vision reported a photo as `people, 3 faces, cluster 4`. The model read the same thumbnail as "cooking class at Sur La Table, three people in aprons actively working," selected it as the social pivot in a five-photo post, and captioned the set "ate well, cooked better." It also rejected all three photos I had deliberately planted as bad, and correctly offered the near-duplicate as a swap rather than including both.

---

## A feature I deleted

The original thesis was that duvro should recognize who appears most often in your camera roll and center the post on them.

I built it, then measured whether it worked. Vision produces "feature prints," vectors you can compare for similarity. Comparing face crops across photos: same-person and different-person distances overlapped almost completely. No threshold separated them.

The reason is that general image feature prints encode lighting, framing, and composition, not identity. Two photos of the same person in different rooms look further apart than two different people photographed the same way.

Three options existed. Use Photos' People albums (private API, no public access). Ship a Core ML face-embedding model (licensing questions around scraped face datasets, plus biometric privacy statutes like BIPA). Or drop it.

I dropped it, and moved the inference to Stage 2: the model infers the recurring subject visually from the thumbnails. The deleted code is gone, not commented out, with the reasoning recorded so it doesn't get rebuilt.

There was also a product problem hiding in the technical one. The largest face cluster in almost any camera roll is the owner's own selfies. A feature built to center posts on the recurring person would have centered every post on selfies.

---

## Three bugs worth writing down

**Memory: 733MB peak against a 400MB ceiling.**

Exporting seven full-resolution photos through a filter chain was using nearly twice the memory iOS allows before it kills the app. The per-photo instrumentation showed the shape of it: photos 1 through 3 cost about 15MB each, photos 4 through 6 cost 100 to 123MB each and never came back down.

The expensive ones were all 24 megapixels. But the diagnostic line that mattered was this:

```
photo 4 clearRenderCaches: 291.2MB -> 291.2MB (Δ+0.0MB)
```

The function whose entire job is freeing memory was freeing nothing, every time. `autoreleasepool` releases objects; it does not touch a shared `CIContext`'s internal render memory, and `clearCaches()` provably wasn't reaching whatever was being retained.

Two fixes: cap render resolution at 2048px on the long edge (Instagram serves carousels at roughly 1080x1350, so nothing visible is lost), and use a fresh `CIContext` per export render that gets deallocated afterward. The cap alone would have hidden the problem rather than solved it, since current iPhones shoot 48MP.

Result: **733MB to 151MB**, with per-photo cost flat regardless of source resolution.

**A parameter that did nothing.**

Every film-grain value in every aesthetic pack was meaningless. The `intensity` parameter adjusted the noise texture's internal contrast, but the composite onto the image had no opacity control, so it always ran at full strength. Setting grain to 12 or 120 produced nearly identical output.

This was invisible until someone looked at a rendered photo and said "the grain is too heavy," and the fix turned out to be structural rather than a tuning change.

**A test tool that would have invalidated an evaluation.**

To test the selection prompt against real photos, I wrote a script that downsamples images with macOS `sips` and posts them to the endpoint.

`sips -Z` strips the EXIF orientation tag without rotating the pixels.

Every portrait iPhone photo would have reached the model sideways. The selections would have been bad, and I would have blamed the prompt. Caught by building a fixture with a known orientation tag and checking the output rather than trusting the resize.

---

## Constraints that shaped the product

**Add-only photo permission.** duvro requests the narrowest photo permission iOS offers: it can add finished photos to your camera roll, and cannot read, browse, or delete your library.

That decision cost a feature. Saving into a named "duvro" album requires `PHAssetCollectionChangeRequest`, which crashes under add-only authorization, and finding an existing album requires read access that add-only doesn't grant. Verified by isolation test: removing album creation entirely made export work.

So v1 saves to the camera roll and the handoff reads "select your 7 most recent photos, tap them in order." Slightly worse UX, and the privacy claim on the welcome screen stays true.

**Preview must match export exactly.** The three-way edit screen shows a downsampled render; export re-renders at full resolution. Those go through one shared crop function, deliberately, because forking it is the easy way to ship an app that exports something the user never approved.

This is also why a planned generative-editing tier has to generate at full resolution and downsample for preview, rather than the reverse: generation is non-deterministic, so a second generation at export time would produce a different image.

---

## Stack

Swift / SwiftUI, Vision, Core Image, PhotosUI. Node serverless functions on Vercel, zero dependencies. Anthropic API with prompt caching on the system prompt. DeviceCheck for per-device entitlement without accounts (ES256 JWT signing, no external storage: Apple holds the bits).

---

## Status

The full flow works end to end: import, curate, edit, review, caption, export, hand off to Instagram. The selection call is live. Nine aesthetic variants exist; three are tuned against reference grades, six are not yet.

The next milestone is a blind test: three people hand-pick seven photos from their own camera roll before seeing duvro's picks, and duvro has to overlap on at least four or be rated equal or better. Nothing ships until that passes.

---

<!-- TODO: add screenshots (vibe selection, three-way edit, showcase collage, final review) -->
<!-- TODO: add a few representative source files -->
