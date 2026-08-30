---
name: keyboard-shortcuts-layouts
description: Use when implementing app keyboard shortcuts (Ctrl/⌘+K palettes, single-key hotkeys) that must work on every keyboard layout — the e.key vs e.code split, why non-Latin layouts (Persian, Arabic, Cyrillic, Greek, Hebrew) silently break key-only matching, modifier normalization, and when NOT to fire.
---

# Keyboard Shortcuts Across Layouts

A shortcut that works on the developer's QWERTY keyboard and is dead for every
user on a non-Latin layout — and nobody files a bug, because to those users the
feature simply doesn't exist. On a Persian layout, pressing the physical K key
with Ctrl held produces `e.key === 'ن'`; `e.key === 'k'` never matches, and
the ⌘K palette is unreachable for the exact bilingual users an RTL-aware app
courted. Same story for Arabic, Hebrew, Greek, Cyrillic.

## The two properties, and what each one means

- **`e.key`** — the *character the active layout produces*. Layout-dependent.
  `'k'`, `'K'`, `'ن'`, `'л'`, `'κ'` are all the same physical key.
- **`e.code`** — the *physical position*, named after its US-QWERTY cap
  (`'KeyK'`, `'Period'`, `'Slash'`, `'Digit1'`). Layout-independent.
- `e.keyCode` / `e.which` are deprecated; don't reach for them.

Neither alone is right:

- **`key` alone** breaks every non-Latin layout (above).
- **`code` alone** breaks remapped Latin layouts: a Dvorak or AZERTY user reads
  "Ctrl+K" in your menu and presses the key that *types* K — which is not
  `code: 'KeyK'`. Their muscle memory follows the letter, not the position.

## The rule: match either, letter first

```js
const k = e.key.toLowerCase() === 'k' || e.code === 'KeyK';
if ((e.metaKey || e.ctrlKey) && k) { e.preventDefault(); open(); }
```

- `key` satisfies everyone whose layout can type the letter (QWERTY, Dvorak,
  AZERTY…). `code` is the fallback that catches layouts where **no keypress
  ever yields a Latin 'k'** — there the physical-position reading is the only
  sensible one, and it is also what those users expect (the Latin cap is
  usually printed on the same physical key).
- `toLowerCase()` — Shift or CapsLock turns `key` into `'K'` and quietly breaks
  an exact-match `=== 'k'`.
- Punctuation shortcuts need the same pair: `e.key === '.' || e.code ===
  'Period'` — symbols move between layouts just like letters do.
- `e.metaKey || e.ctrlKey` covers ⌘ on macOS and Ctrl elsewhere in one check.
  Advertise it per-platform in the UI (`⌘K` / `Ctrl+K`), but accept both.

## When not to fire

Guard every global shortcut with all three, or it will eat real typing:

1. **Focus in an editable target** — `input`, `textarea`, `select`,
   `[contenteditable]` (check `closest('[contenteditable]')`, not just the
   target). Modifier-chord shortcuts (Ctrl+K) may be exempted deliberately;
   bare-key shortcuts (`/`, `.`, `j`/`k`) never.
2. **IME composition** — skip when `e.isComposing` (CJK and other composed
   input; the events you see mid-composition are not real keystrokes).
3. **Already-claimed chords** — `preventDefault()` only after your match, so
   unmatched keys keep their browser behavior.

## Displaying the right label

`navigator.keyboard.getLayoutMap()` can translate a `code` back to the active
layout's character for display — Chromium-only; fall back to the US label.
Never display the raw `code` string (`'KeyK'`) to users.

## Testing

A layout bug is invisible on the author's machine by construction. Test by
switching the OS input source (e.g. Persian) and pressing the physical chord —
or synthesize it: `new KeyboardEvent('keydown', { key: 'ن', code: 'KeyK',
ctrlKey: true })` in a unit test pins the fallback path forever.
