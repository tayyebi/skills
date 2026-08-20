---
name: multi-calendar-datetime
description: Use when an app must display and accept dates in more than one calendar system (Gregorian + Jalali/Hijri/…) — universal instant storage, Intl-only calendar math, calendar-aware date fields and grids, and the timezone/day-boundary traps. Distilled from shipping end-to-end Jalali support in a multi-tenant app.
---

# Multi-Calendar Datetime

Support multiple calendar systems (Gregorian, Jalali/Persian, Hijri, …) end to end — reading AND typing — without a date library, without corrupting storage, and without the app disagreeing with itself about what day it is.

## When to use

Any app serving users whose civil calendar is not Gregorian (Iran, Afghanistan, much of the Middle East, Ethiopia, Thailand…). Especially before the first non-Gregorian tenant signs up: half-done calendar support (display-only) is worse than none, because users read ۱۴۰۵ everywhere and are then forced to *type* Gregorian.

## The one governing rule

**A calendar is a way of SHOWING a day, never a way of storing one.**

- Every stored temporal value is a universal instant (`timestamptz`) or a Gregorian ISO `YYYY-MM-DD` day key. No Jalali ever reaches the database, an API parameter, a cache key, a CSV cell key, or a URL.
- Every date input/picker **emits Gregorian ISO** regardless of what it displays. This makes multi-calendar support cheap: routes, SQL, exports, and lexical `min`/`max` comparisons never change.
- Display resolves through ONE formatting context: `{ readerLocale, calendar, timeZone, dateFormat, weekStartsOn }` — policy fields from the tenant/org, digits and language from the reader. Build it in exactly two places: a client hook and a server-side `dateContextForOrg(orgId, readerLocale)` twin (exports, bots, emails, share pages need the server one).

## Calendar math with Intl only — no date library

`Intl` already knows every CLDR calendar via the BCP-47 `-u-ca-` extension (`en-US-u-ca-persian`). You don't need moment/dayjs/jalaali-js, and (as of 2026) you can't rely on `Temporal` in Node runtimes.

- **Two-tag rule, never mixed:** a *math* tag is always `en-US-u-ca-<calendar>` (ASCII digits, `Number()`-parseable parts); a *display* tag is the reader's locale (renders ۱۴۰۵). Never `parseInt` a display formatter's output.
- **Instant → calendar date** is direct: `formatToParts` with the math tag + target zone.
- **Calendar date → instant** (the inverse Intl does not provide): estimate the Gregorian instant (`gy ≈ jy + 621` for Jalali, +/− a couple of days), then step/binary-search noon-UTC candidates comparing formatted parts until they match, then correct ±1 day for the zone. Normalize results to **noon in the target zone** — a noon anchor has ~11h of slack so no DST transition can flip the day.
- **Never tabulate leap rules.** Derive month lengths by probing Intl (format the 1st of next month, step back one day). Esfand is 29 or 30; hardcoding the 33-year cycle is a bug factory, and probing works unchanged when you add the next calendar.
- **Memoize formatters** in a `Map` keyed by `tag|zone|options` — a 42-cell month grid must not construct 42 `Intl.DateTimeFormat`s per render.
- Request `era` in math formatters and reject unexpected eras (AP vs AD) rather than silently mis-mapping.

## The traps (each one shipped as a real bug)

- **Bare locale tags choose a calendar for you.** `toLocaleDateString('fa-IR')` uses the *Persian* calendar by default in ICU — a "deliberately Gregorian" chart with `fa-IR` labels silently shows Jalali month names over Gregorian columns. When you mean Gregorian-in-Persian-script, say `fa-IR-u-ca-gregory`.
- **`toISOString().slice(0,10)` is the UTC day**, not anyone's "today". Every "today"/"last 7 days"/date-range preset computed this way is off by one for part of every day in any non-UTC zone. Compute today from the tenant zone (via the calendar-math layer), emit Gregorian ISO.
- **`new Date(y, m, d)` and `.getDay()` are host-local.** Server TZ is unpinned in containers; browser TZ is the visitor's. Both are wrong inputs for tenant-day logic.
- **Wall-clock future events need a per-row IANA zone.** "09:00" is an intent about a *place*: store `starts_at timestamptz` + `time_zone text` per row. It makes rescheduling DST-correct, lets other-zone viewers read "09:00 (Asia/Tehran)", and stops a tenant settings change from relabelling history. Compute wall→instant **in the database** (`(local_ts) AT TIME ZONE tz`), so backfill and runtime writes share one DST engine; add day-crossing intervals to the *local* timestamp **before** `AT TIME ZONE`.
- **Stored/dispatched text freezes at write time.** Notification bodies, timeline entries, .ics files, emails: format them zone-anchored (and label the zone) when written — they can't be re-rendered later.
- **`Intl.RelativeTimeFormat` once.** Hand-rolled "N days ago" helpers multiply (we deleted seven) and are invariably typed `isRtl: boolean`, so the third language reads English. One helper taking a real locale; `numeric: 'auto'` gives yesterday/دیروز/gestern free.
- **Boolean-locale props (`fa: boolean`) are debt** — guard them with a ratchet test (an exact count that may only shrink) so new code takes a `Locale`.

## Calendar-aware UI

- **Date field:** displays and parses in the tenant calendar, emits ISO. Parsing: normalize native digits to ASCII → split on `[/\-.،\s]` → map through the tenant's field order → inverse-search to ISO → clamp to min/max. Resolve 2-digit years to the nearest century *of the display calendar* ('04' → 1404). Add paste heuristics both ways (a 2026 pasted into a Jalali tenant is Gregorian; a 1404 pasted into a Gregorian tenant is Jalali).
- **Month grid / heatmap:** never build from month-length tables — *day-walk* with the calendar-aware add-days (which owns clamping and leap rules), keying every cell by Gregorian ISO so data lookups and drill-down links are calendar-blind. Honour `weekStartsOn` (Saturday for Iran); take weekday headers from Intl (`narrow` width — Persian `short` names collide at picker width).
- **RTL:** logical properties only. A `grid-auto-flow: column` grid auto-flips under `dir=rtl` — but absolutely-positioned overlays (month tick labels) must use `inset-inline-start`, not `left`, or every label sits over the wrong column. Mirror chevrons (`rtl:rotate-180`).
- **Locale-aware digits break `parseInt` on your own output** — e.g. "label every 6th hour" via `parseInt(formattedHour)` hides all labels in Persian; use the array index.
- A year in calendar A straddles two years of calendar B: a "fetch by Gregorian year" API serving a Jalali-year view needs two fetches merged client-side (cheaper than changing the API/cache).

## Testing that actually catches it

- **Round-trip sweep:** every day of a decade, `iso → calendarDate → iso`, in an eastern (UTC+3:30) *and* western (UTC−5) zone. This is the safety net for the inverse search.
- Extreme zones (UTC+14 Kiritimati, UTC−11) pin the ±1-day correction; leap probes pin Esfand 29/30 in adjacent years.
- Wall-clock DST cases: the same local time a week apart across a transition differs by 167/169 hours, not 168; a cross-midnight span on a DST night is 90 real minutes, not 150.
- Migration backfills: `ADD COLUMN … DEFAULT` stamps existing rows, so a later `WHERE col IS NULL` backfill matches nothing — drive the backfill off the *data* column being NULL, set the default afterwards. Test the backfill against a hand-built pre-migration schema on real Postgres, twice (idempotence), under ≥2 host TZs.
- Test the *positive* case of any SQL-embedded regex from a template-literal language: `\d` in a JS template cooks to `d` and silently matches nothing.

## Pitfalls (summary)

- Display-only calendar support: users read Jalali but must type Gregorian.
- Storing calendar-local dates, or letting the display calendar leak into keys/APIs.
- Reconstructing instants by string concatenation at read time — every reader disagrees about the zone suffix.
- Trusting the server's `TZ` for anything.
- Tabulated leap years; `parseInt` on localized output; bare `fa-IR`-style tags when Gregorian is intended.
- Charts/heatmaps "temporarily excepted" from calendar-awareness — the exception rots into a visible bug.
