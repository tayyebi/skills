---
name: multi-channel-notifications
description: Use when an app needs to notify users across more than one channel — a single dispatch that fans out to in-app inbox, email, SMS, and chat apps while honoring per-user opt-out preferences. Covers the preference matrix and provider abstraction.
---

# Multi-Channel Notifications

One call to notify a user; the system decides which channels to deliver on based on their preferences.

## When to use

Any app that sends more than the occasional email. The moment you have both an in-app bell *and* email, unify them behind one dispatcher instead of sprinkling `sendEmail()` calls everywhere.

## Core pattern

**A single `notify(user, event, payload)` entry point** that fans out:

```
notify(user, 'candidate.assigned', {...})
   ├─ in-app inbox row (always, for the bell/unread count)
   ├─ email        (if user.prefs[event].email !== false)
   ├─ SMS          (if user.prefs[event].sms === true)
   ├─ chat app     (if linked + enabled)
   └─ webhook      (tenant-level outbound event)
```

- **Preference matrix**: `notification_prefs` is a JSONB map of `event → { email, sms, chat }` booleans, with sensible defaults (in-app on, email on, SMS off). A settings UI renders this as a grid.
- **Default-on with opt-out** is usually right for transactional notifications; default-off for anything marketing-adjacent.
- **Channel = provider abstraction.** Each channel has a common interface (`send(recipient, message)`), so SMS can have multiple interchangeable providers behind it (see below).

## Provider abstraction (per channel)

For SMS/email especially, define a provider interface and register concrete providers (Kavenegar/Twilio/SMS.ir for SMS; SMTP for email). Select by config. This lets an operator swap providers without code changes and lets you fall back if one is down.

## Implementation notes

- **In-app inbox**: `notifications(id, user_id, event, body, read_at, created_at)` drives the bell badge / unread count. Support @-mention fan-out (notify every user matching a role or named in a comment) and "watchers" (users subscribed to an entity).
- Build the message body once, render per-channel (markdown → HTML for email, plaintext for SMS).
- Make dispatch **non-blocking** to the request — enqueue as a job (see `background-jobs-scheduling`) so a slow SMTP server doesn't stall the user's action.
- Digest/report notifications (daily/weekly) reuse the same channel abstraction.

## Pitfalls

- Sending on a channel the user muted — always consult prefs *inside* dispatch, not at call sites.
- No unread/read state — the inbox needs `read_at`, and marking-read should be idempotent.
- Blocking the request thread on email/SMS latency.
- Missing defaults for brand-new event types (treat unknown-event prefs as the default, not "off").

## Checklist

- [ ] Single `notify()` fan-out entry point
- [ ] JSONB preference matrix with defaults, editable in settings
- [ ] Per-channel provider interface (swappable, with fallback)
- [ ] In-app inbox with unread count + read state
- [ ] Dispatch runs async (queued), not inline
- [ ] @-mention / watcher fan-out
