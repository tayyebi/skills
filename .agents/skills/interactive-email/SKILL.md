---
name: interactive-email
description: Use when outbound email should be actionable in the inbox — calendar invites recipients can accept (RFC 5545/5546), Gmail JSON-LD event cards and action buttons, signed one-click action links (RSVP, unsubscribe per RFC 8058), auto-reply suppression, and thread grouping. Covers the wire formats, the header set, and the token pattern for public action endpoints.
---

# Interactive Email

Make transactional email actionable inside the mail client instead of a wall of prose: an event the recipient adds (or RSVPs to) in one click, buttons that hit your API without a login, and an unsubscribe the inbox itself renders.

## When to use

Any app that emails people about *events* (meetings, interviews, bookings, deadlines) or sends *automated* sequences. If a human is expected to act on the email — attend, confirm, decline, opt out — encode the action in standards the client understands rather than only prose + links.

## The four layers (independent — adopt any subset)

### 1. iCalendar invites (RFC 5545 format, RFC 5546 semantics)

Attach a single-VEVENT `VCALENDAR` to the email. With nodemailer this is the `icalEvent: { method, filename, content }` option — it wires the right MIME structure (`text/calendar; method=X` alternative + attachment) for you.

Method selection matters more than anything else:

- **`METHOD:REQUEST`** — a real invitation. Client shows Accept/Decline UI. Requires `ORGANIZER` and an `ATTENDEE` line for the recipient (`PARTSTAT=NEEDS-ACTION;RSVP=TRUE`).
- **`METHOD:CANCEL`** — retracts a previously sent UID. Same `UID`, `SEQUENCE` bumped, `STATUS:CANCELLED`.
- **`METHOD:PUBLISH`** — "here's an event", no RSVP round-trip. Right for FYI copies (e.g. internal staff) where the recipient isn't an attendee you track.

Rules that bite if ignored:
- **Stable UID per logical event** (`<entityId>@yourapp`) — reuse it across the invite, updates, cancellation, and any subscription feed, or clients duplicate the event instead of updating it.
- **`SEQUENCE` increments on every re-issue** of a UID (update/cancel), else the client ignores the newer message.
- **CRLF line endings** and **75-octet line folding** (continuation lines start with one space).
- Escape `\ ; , \n` in text values; `DTSTAMP` is a full `YYYYMMDDTHHMMSSZ` timestamp.
- An embedded `VALARM` (`TRIGGER:-PT30M`, `ACTION:DISPLAY`) survives import in most clients — free reminder.
- RSVP replies travel as `METHOD:REPLY` emails to the ORGANIZER address — you won't see them unless you parse inbound mail. If you don't, add your own HTTP confirm/decline links (layer 3) and treat the ICS buttons as best-effort.

### 2. schema.org JSON-LD (Gmail "email markup")

Inject `<script type="application/ld+json">` blocks into the HTML part (before `</body>`). Gmail parses them and renders structured UI:

- **`EventReservation`** → calendar card with event details + add-to-calendar. Works for any SPF/DKIM-passing sender; no registration.
- **`EmailMessage` + `ViewAction`** → a "go-to" button (Join meeting, View booking) next to the subject line. Google requires sender registration for the inline button to show — embedding it anyway is harmless (ignored when unregistered), so always include it and let deliverability config decide.
- **`EmailMessage` + `ConfirmAction`** (with `HttpActionHandler`) → one-click confirm that POSTs to your URL from the inbox. Handler must be unauthenticated-POST-safe and idempotent — exactly the signed-token endpoint from layer 3.

Escape `<` inside the JSON (`<`) so user-supplied strings can't close the script tag. Belt-and-braces: the email body should read fine when none of this renders — JSON-LD is progressive enhancement.

### 3. Signed action links (login-free, no token table)

Public endpoints that mutate state from an email click need capability tokens, not sessions. HMAC beats a DB token table for this:

```
token  = base64url(entityId) + '.' + base64url(HMAC-SHA256(secret, purpose + ':' + entityId))
verify = recompute; timingSafeEqual; return entityId or null
```

- **Bind the purpose into the MAC** (`'unsubscribe:' + id` vs `'rsvp:' + id`) so a token minted for one action can't be replayed against another endpoint.
- Endpoints are org/tenant-agnostic — the entity id resolves the tenant.
- **GET renders a confirmation page with a POST form; only POST mutates.** Link prefetchers and safe-link scanners follow GETs — a GET that unsubscribes/declines will fire spuriously. (RFC 8058 clients POST directly; humans get the page.)
- Idempotent: repeating a choice is a no-op; changing it overwrites.
- On action: write the state change, drop an audit/timeline entry, notify the humans who care, fire a webhook. Show a plain self-contained HTML page (inline styles) as the response.

### 4. Standards headers on automated mail

Every machine-generated send should carry:

- `Auto-Submitted: auto-generated` (RFC 3834) + `X-Auto-Response-Suppress: All` (Outlook) — stops out-of-office loops and auto-replies.
- `List-Unsubscribe: <https://…>` (RFC 2369) + `List-Unsubscribe-Post: List-Unsubscribe=One-Click` (RFC 8058) — inbox-rendered unsubscribe; Gmail/Yahoo require it for bulk senders. Point it at a layer-3 signed endpoint that flips a per-recipient `email_opt_out` flag your sender honors. Scope: automated sequences only — not personal 1:1 mail from a human.
- **Deterministic `Message-ID` threading**: give the first email about an entity a computed id (`<interview-{id}@{from-domain}>`); later emails about the same entity set `In-Reply-To`/`References` to it so clients group the conversation. No storage needed.

## Integration notes

- Thread the event payload (title/date/times/location/link + entity id) into whatever queue/job executes the send, pinned at schedule time — the send must not re-read mutable state to build the invite it promised.
- Suppression flags (`email_opt_out`) belong on the recipient row and are checked *in the sender/runner*, marking the job skipped (not failed) with a reason — same shape as any other per-recipient suppression (e.g. silent-reject).
- Dedupe guards keyed "once per recipient per rule" must switch to "once per *event* per rule" for event-scoped emails, or the second meeting with the same person never gets an invite.
- Reminder triggers invert delay semantics: the rule's delay is *lead time before start*. A cron scan selects upcoming events within the window with a `NOT EXISTS` guard on (rule, eventId) for exactly-once.
- All-in-one send API shape: extend the mailer input with `calendarEvent {method, content}`, `jsonLd []`, `autoSubmitted`, `listUnsubscribeUrl`, `messageId/inReplyTo/references`, and translate in one place.

## Pitfalls

- Sending `REQUEST` without an `ATTENDEE` for the recipient — some clients show nothing actionable. Use `PUBLISH` when you can't name the attendee.
- Regenerating a random UID per send — updates/cancels duplicate instead of replacing.
- One-click unsubscribe on a GET — scanners opt users out silently.
- Testing only in one client: Gmail, Outlook (desktop + web), and Apple Mail disagree the most on ICS rendering; the JSON-LD card is Gmail-only.
- Forgetting that the JSON-LD/ICS duplication is intentional — ICS covers non-Gmail clients, JSON-LD upgrades Gmail; both must describe the same event.

Related: [[multi-channel-notifications]] for the dispatch/preferences layer, [[background-jobs-scheduling]] for the queue that executes sends, [[share-links-tracking]] for the adjacent signed-URL pattern.
