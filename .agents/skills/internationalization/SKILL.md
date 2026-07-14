---
name: internationalization
description: Use when an app must support more than one language or a right-to-left script — a translation dictionary, per-user and per-tenant locale, RTL layout, and locale-aware digit/number handling. Covers key organization and the digit-normalization trap.
---

# Internationalization & Localization

Support multiple languages and scripts (including RTL like Arabic/Persian/Hebrew) without hard-coding English into the UI.

## When to use

Any app with users outside a single language market — and *especially* before launch, because retrofitting i18n into a codebase full of hard-coded strings is expensive.

## Core pattern

- **Translation dictionary**: a key → `{ en, fa, … }` map. UI code references stable keys (`t('candidate.save')`), never literal strings. Keep keys namespaced by feature.
- **Locale resolution order**: explicit user preference (`user.ui_prefs.locale`) → tenant default (`site_settings.localization`) → `Accept-Language` → fallback locale. Always have a fallback so a missing translation degrades to a readable string, not a blank.
- **Content vs. UI**: UI chrome uses the dictionary; user-authored *content* (a job description) needs its own per-record translations field (`translations jsonb`) — don't force content through the UI dictionary.

## RTL

- Drive direction from the locale (`dir="rtl"`). Use **logical CSS properties** (`margin-inline-start`, `padding-inline-end`, `text-align: start`) so one stylesheet works both directions — avoid hard `left`/`right`.
- Test the whole UI in RTL; icons, chevrons, and progress directions often need mirroring.
- Bundle a font that covers the target script (e.g. Vazirmatn for Persian) and self-host it.

## Locale-aware numbers & digits (the trap)

- Scripts like Persian/Arabic use their own digit glyphs (`۱۲۳` / `١٢٣`). **Users will type them into number/phone/date fields.**
- **Normalize on input**: fold non-ASCII digits to ASCII before validation, storage, and comparison — otherwise a phone number "matches" visually but fails an equality check or a DB lookup.
- **Localize on output**: render numbers/dates in the user's locale for display.
- Format numbers, currency, and dates with `Intl.*` (or equivalent), not manual string building.

## Pitfalls

- Hard-coded strings scattered in components — you can't translate what isn't keyed.
- String concatenation for sentences (`"You have " + n + " items"`) — breaks grammar/word-order across languages; use parameterized messages with pluralization.
- Physical `left`/`right` CSS — breaks RTL.
- **Not normalizing input digits** — the single most common i18n bug in Persian/Arabic apps; causes phantom search/validation failures.
- Assuming Latin sort/casing rules for all locales.

## Checklist

- [ ] Key-based translation dictionary with fallback locale
- [ ] Locale resolution: user → tenant → header → default
- [ ] Separate per-record translations for user content
- [ ] RTL via logical CSS properties; script-covering font
- [ ] Digit normalization on input, locale formatting on output
- [ ] Parameterized, pluralization-aware messages
