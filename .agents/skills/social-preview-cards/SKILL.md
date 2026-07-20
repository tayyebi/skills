---
name: social-preview-cards
description: Use when shared links must render as branded preview cards (Open Graph / Twitter cards) generated per page rather than a single static image — server-side image rendering, content-addressed caching so cards never go stale, and the silent-failure traps in rasterizing text (especially non-Latin scripts).
---

# Social Preview Cards

Generate a per-page preview image so a shared link renders as a designed card instead of a bare URL or a cropped product photo. The hard parts are not the design — they are text rendering (which fails silently) and cache invalidation (which fails invisibly).

## When to use

- Links to your product get shared into chat apps and social feeds, and currently show nothing, a generic logo, or a raw uncropped asset.
- Card content must reflect live data — a title, price, author, rating, or item count that changes.
- **Skip it** when every page would show the same image: a single static asset referenced site-wide is strictly simpler and cheaper. Generation earns its keep only when the card is per-entity.

## Core pattern

Three decisions, in order.

**1. Composition: background + measured layers.** Build the card as a vector background (gradients, rules, icon shapes) with raster layers composited on top (rendered text runs, photos). Measure text *before* placing it, then position blocks as a stack — card content varies (a record with no rating, no badge and a short title) and fixed offsets leave a well of empty space or overflow.

**2. Rendering engine: pick for script coverage, not ergonomics.** See the section below — this is the decision most likely to be made wrongly.

**3. Caching: content-addressed, generated on demand.** The public URL is stable and un-hashed (`/og/product/<slug>`); the hash lives in the *storage key* behind it.

```
key = hash(card_content + theme + template_version)
GET /og/<target>:
  resolve target -> card content (one cheap query)
  compute key
  HEAD key -> hit? stream stored bytes
           -> miss? render, store, serve
  ETag = the hash
```

Invalidation becomes implicit: rename the entity, change a brand colour, or edit a template, and the key changes, so the next request renders fresh. **Nothing is ever purged and a stale card cannot be served.** Superseded objects are garbage a housekeeping sweep can reclaim; they are never *wrong*. Contrast with pre-generating every card in a batch job, which needs an invalidation trigger on every field the card displays — the thing everyone forgets. See `content-addressed-file-storage` for the general addressing pattern.

Version the template explicitly (`TEMPLATE_VERSION`) and fold it into the hash, or a design change leaves every existing card on its old key forever.

## Text rendering fails silently — the central warning

Image rendering returns a valid image of the *wrong thing*. There is no exception, no error log, and a plausible byte count. **Typechecks, unit tests and exit codes cannot see it.** Render a sample and look at it — with content that stresses the hard case (mixed-script text, a very long title, a missing photo).

Concrete traps, each of which produces a successful render of broken output:

- **Wrong font format.** Web font formats (`.woff2`) are typically unreadable by the system font stack used for server-side rasterizing. Pointing at one silently falls back to a default face — non-Latin glyphs become tofu boxes (`□□□`) while Latin looks fine, so a Latin-only smoke test passes.
- **Missing alpha flag.** Text rasterizers often default to opaque output; the layer then composites as a solid black rectangle over the card. Verify the layer has an alpha channel.
- **Options the library accepts but ignores.** A flag that isn't in the installed version is frequently dropped without complaint rather than rejected. Check the option exists in *that version's* source before trusting it — passing a plausible-looking `rtl: true` to a library with no such option is indistinguishable from it working, until an edge case exposes it.
- **Glyphs the font lacks.** Brand faces often have no `★`, `✓`, or arrow glyphs. Draw those as vector paths in the background layer; typesetting them yields tofu.
- **Translucent gradient stops.** A semi-transparent stop composited against a transparent canvas reads far more saturated once flattened than it does in a browser preview. Mix to explicit solid colours instead.

## Non-Latin and bidirectional text

If any card text can be Arabic, Persian, Hebrew, Urdu, or an Indic script, script support drives the engine choice:

- The renderer must do **shaping** (letters change form by position and join) and **bidi reordering**. Popular JSX-to-image renderers built for the web ecosystem often do *neither* — they are excellent for Latin and produce disconnected, wrongly ordered letterforms otherwise. Verify with a real string before building on one.
- Engines backed by a full text stack (HarfBuzz/Pango, or a headless browser) shape correctly.
- **Pin the base direction explicitly.** Renderers commonly infer direction from the first strong character, so a title beginning with a Latin brand name flips the whole line's base direction and throws that run to the wrong edge. Wrap RTL runs in Unicode isolates (`U+2067` … `U+2069`) — the raster-side equivalent of the CSS `unicode-bidi: isolate` in `rtl-bidi-layout`.
- Locale-aware formatting still applies: convert digits and use the locale's decimal separator, which a digit-only mapping will miss. See `internationalization`.

## Serving

- **Public and unauthenticated.** Crawlers fetch these with no session. Everything on the card is already public on the corresponding page — never put anything else on one.
- Emit the image URL as **absolute**. Crawlers don't resolve relative URLs reliably.
- Feed the same URL to **both** the Open Graph and Twitter image tags; the Twitter tags don't inherit from Open Graph in every framework's metadata layer.
- Declare width/height/alt alongside the URL.
- **An empty image array can suppress a site-wide fallback.** Emitting "no images" is not the same as omitting the field — a page with no entity photo can end up sharing with no image at all. Fall back to the generated card, then to a static default.
- `Cache-Control` with a short browser TTL and a long shared/CDN TTL; set the ETag to the content hash so revalidation is a cheap 304.
- Degrade, never fail: if object storage is unavailable, render and serve inline (uncached). A broken preview is worse than an uncached one.
- Keep an authenticated *preview* endpoint separate from the public one if you offer an editor — it renders uncommitted theme overrides and must never write to the cache.

## Pitfalls

- **Reusing a heavyweight "get full entity" query.** A crawler hitting a cold card shouldn't fan out into a dozen joins for reviews and related items the card never shows. Write one narrow query.
- **Unbounded text.** A long title must be measured and truncated on a word boundary, then re-rendered — cropping the rasterized layer slices glyphs in half and breaks shaping.
- **Composite bounds.** Most imaging libraries throw if a layer lands outside the canvas; clamp offsets and drop oversized layers rather than failing the whole card.
- **Untrusted values interpolated into a vector document.** Theme colours and text reach the renderer from settings or a query string; validate colours against a strict pattern and escape text, or you have markup injection into the generated document.
- **Fonts dropped by the build.** Bundlers only trace files that are imported. Font files read from disk at runtime must be explicitly included in the deployment artifact, or they vanish in production only — and the tofu appears where nobody is looking.
- **Cache lookups for keys that are expected to be missing.** A cold render asks storage for something that isn't there on every first request; verify the *miss* path's latency, not just the hit. Prefer a cheap existence check (HEAD) before opening a stream.
- Cards that leak non-public data, or that a crawler can be made to render for an unpublished/hidden entity — resolve visibility exactly as the page does.

## Checklist

- [ ] Renderer verified to shape and reorder every script the content can contain
- [ ] A card rendered and **visually inspected** with mixed-script, long, and missing-photo content
- [ ] Fonts in a system-readable format, present in the deployed artifact
- [ ] Base direction pinned; digits and separators locale-formatted
- [ ] Icons as vector paths, not font glyphs
- [ ] Content-addressed key incl. an explicit template version; ETag = hash
- [ ] Narrow purpose-built content query
- [ ] Text measured, clamped on word boundaries; composite offsets bounded
- [ ] Colours/text validated before entering the vector document
- [ ] Absolute URLs on both OG and Twitter tags, with width/height/alt
- [ ] Empty-image-array case falls back rather than suppressing
- [ ] Graceful degradation when storage is unavailable; miss-path latency measured
