---
name: rtl-bidi-layout
description: Use when a UI must render correctly in a right-to-left script (Arabic, Persian/Farsi, Hebrew, Urdu) — logical CSS properties over physical left/right, direction-aware icon mirroring, bidirectional (bidi) text handling for mixed-script content, and testing under a real RTL locale. Covers the layout/CSS-mechanics half of RTL support; see `internationalization` for translation dictionaries, locale resolution, and digit normalization.
---

# RTL & Bidirectional Layout

Building layout that flips correctly for right-to-left scripts without maintaining a second, drifting stylesheet. The mechanism is CSS logical properties plus a small set of deliberate exceptions for content that must never mirror.

## When to use

- Any product that must support Arabic, Persian/Farsi, Hebrew, or Urdu — even if today it's "just one RTL locale," build the mechanism generically.
- Building a shared component library — bake logical properties into the base components now; retrofitting them one RTL bug report at a time is far more expensive.

## Core pattern: logical over physical

CSS logical properties describe position relative to text flow (`start`/`end`, `block`/`inline`) instead of screen geography (`left`/`right`). Written once, they flip automatically with direction — no duplicate ruleset needed:

| Physical (avoid in shared layout) | Logical (prefer) |
|---|---|
| `margin-left` / `margin-right` | `margin-inline-start` / `margin-inline-end` |
| `padding-left` / `padding-right` | `padding-inline-start` / `padding-inline-end` |
| `border-left` / `border-right` | `border-inline-start` / `border-inline-end` |
| `left` / `right` | `inset-inline-start` / `inset-inline-end` |
| `text-align: left/right` | `text-align: start/end` |
| `float: left/right` | `float: inline-start/inline-end` |

```
/* one ruleset, correct in both directions */
sidebar {
  inset-inline-start: 0;
  border-inline-end: 1px solid border-color;
}
main-content {
  margin-inline-start: sidebar-width;
}
```

Reach for a physical property only when the value is genuinely direction-independent (e.g. a fixed-width spacer that must sit at a specific screen edge regardless of script) — that's the exception, not the default.

## Direction strategy

- Drive `direction` from an explicit language/locale attribute at the document or component root — never infer it implicitly from content or guess from a `dir` attribute set once and forgotten. See `internationalization` for how that locale is resolved (user preference → tenant default → header → fallback).
- Scope RTL-specific overrides that logical properties can't express — a different font, hand-tuned spacing for a denser script — under that same language attribute, so they travel with the language rather than living behind a separate flag that can fall out of sync.
- A page can be RTL overall and still contain an LTR block (a code sample, a third-party embed) or vice versa; direction is a property of a subtree, not just the document.

## Mirroring: not everything flips

Two different categories of icon, and they behave differently:

- **Direction-encoding icons** — back/forward chevrons, "next" arrows, progress indicators — mirror, because they encode reading/navigation direction.
- **Meaning-encoding icons** — a play button, a checkmark, a clock face, a company logo — do not mirror; flipping them changes or destroys their meaning.

```
/* direction-bearing icon: flips with the page */
[dir="rtl"] .icon-next { transform: scaleX(-1); }

/* meaning-bearing icon: explicitly excluded from mirroring */
.icon-play { transform: none; }
```

Numerals, phone numbers, URLs, code snippets, and embedded runs of LTR-language text stay left-to-right even inside an RTL page — force the direction explicitly rather than letting the bidi algorithm decide, since a phone number that "looks" mirrored is unreadable.

## Bidi text handling

- Isolate embedded runs of opposite-direction text (a Latin brand name inside an Arabic sentence, a mixed-script product title) with `unicode-bidi: isolate` (or `plaintext` for runs whose direction should be detected from their own content) so punctuation and word order don't get silently reordered around the boundary.
- Default each input's direction to the type of content it expects, independent of the surrounding page direction — a phone/email/URL field stays LTR even embedded in an RTL form; a free-text field matches the page direction.

```
input[data-content="phone"],
input[data-content="email"],
input[data-content="url"] { direction: ltr; text-align: left; }
```

## Testing in an RTL locale

- Don't validate RTL support from a mirrored screenshot alone — run the actual UI against a real RTL locale with real strings (not lorem ipsum): RTL text length, word-breaking, and line-wrapping behave differently enough that placeholder text hides real bugs.
- Fold RTL into the same automated visual-regression and accessibility passes as LTR (see `testing-accessibility`, `testing-browser-ui`) rather than a manual one-off check before a launch — a layout regression in RTL is exactly as real as one in LTR and should fail the same way.

## Pitfalls

- Physical `left`/`right` properties anywhere in shared layout — the single most common source of RTL bugs, and the one logical properties eliminate outright.
- Mirroring content that shouldn't mirror — numerals, phone numbers, third-party logos, a clock icon — which confuses users more than it helps them.
- Treating `dir="rtl"` as sufficient on its own — direction and language are related but distinct, and a subtree can legitimately need one without the other.
- Testing only with lorem-ipsum or a machine-translated placeholder — misses wrapping, truncation, and width bugs that only show up with real script text.
- Maintaining a parallel `rtl.css` (or `[lang="xx"]`-scoped duplicate of every rule) instead of logical properties — it drifts from the base stylesheet the moment either one is edited without the other.

## Checklist

- [ ] All shared layout uses logical properties (`inline-start`/`end`, `block-start`/`end`); no bare `left`/`right` outside deliberate exceptions
- [ ] Direction driven by an explicit language/locale attribute, not inferred or hardcoded
- [ ] Direction-bearing icons mirrored; meaning-bearing icons and numerals/URLs/code left untouched
- [ ] Bidi isolation applied to mixed-script content; content-typed inputs (phone/email/URL) forced LTR regardless of page direction
- [ ] RTL covered by automated visual/accessibility testing with real RTL text, not a manual pre-launch pass
- [ ] No separate/parallel RTL stylesheet to keep in sync
