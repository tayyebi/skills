---
name: mail-client-sync
description: Use when mirroring a remote mailbox (IMAP) or any remote store you do not own into a local database — the mirror + write-back op queue split, leased and budgeted sync runs, folder fairness under a budget, the UID-renumber trap, and honest staleness reporting. Also the protocol traps that only show up in production.
---

# Mail Client Sync

A mail client is not a view over IMAP. It is a **local mirror of a store somebody else owns**, kept roughly current by a job, plus a **queue of commands you still owe that store**. Every hard problem in the thing comes from that split, and almost none of them come from the mail protocol itself — so most of this transfers to any remote system you mirror and write back to (a CRM, a ticketing API, a file host).

## When to use

- You need to search, join, thread, filter, or report over remote data. You cannot do any of that over a protocol that answers one folder at a time.
- The remote store is authoritative and changes without telling you — somebody read a message on their phone between two of your runs.
- Your UI must write back (read, flag, move, delete, send) and must not block on the remote server to do it.

Complements: `background-jobs-scheduling` (what calls the sync), `idempotency-keys` (the write-back queue's cousin), `outbound-webhooks` (what fires when new rows land), `structured-logging-audit` (how you find out fairness is broken).

## Core split: mirror for reads, op queue for writes

```
server ──(sync run)──▶ mirror tables ──▶ UI reads only from here, always
  ▲
  └──(op runner)────── op rows ◀── UI writes here, and returns immediately
```

Two rules that everything else hangs off:

- **The UI never talks to the remote server.** A click writes a local row *and* an op row, then returns. The page is correct instantly; the server catches up. A UI that awaits IMAP is a UI that hangs whenever the mailbox does.
- **Drain the op queue at the START of a sync run, before reading anything.** Otherwise the run reads back the pre-op state and paints the user's own change as undone — the flag flickers off, then on again two minutes later.

## The sync run: leased, budgeted, checkpointed

There is no long-lived worker. Cron calls one function every couple of minutes, "Sync now" calls the same one, and both go through the same lease so two runs never overlap on one account:

```
token = UPDATE accounts SET lock_until = now()+3min, lock_token = $t
          WHERE id = $a AND (lock_until IS NULL OR lock_until < now())
          RETURNING id                       -- no row = someone else has it
```

A run has a **wall-clock budget** (~80s) and stops where it runs out. That is what makes an initial backfill of a decade-old mailbox survivable: each folder checkpoints its cursor, the next run carries on, nobody waits.

Per folder, in this order — and the order is load-bearing:

1. **Generation check** (IMAP: `UIDVALIDITY`). See the renumber trap below.
2. **Reconcile** the window you already have: flags changed, messages expunged.
3. **New items** above the cursor.

Reconcile *before* new mail, or a message that arrived and was read on a phone between two runs comes in unread and then flips.

## Fairness is the bug you will actually ship

This is the least obvious lesson and the one that cost the most. **A budget plus a fixed order means the folders at the back are not synced late — they are never synced at all.** Production logged "3 folders" for the same mailbox run after run while a busy inbox backfilled; Sent sat fourth in a fixed order and stayed empty forever.

It is worse than a clock problem, because **a run also ends when the connection drops**, and the connection drops in the big busy folder. Pinning the inbox first meant every run died inside it and reached nothing else — "1 folder" per run, indefinitely.

The ordering that actually works:

```sql
ORDER BY (last_synced_at IS NULL) DESC,   -- never synced: ahead of everything
         (special_use = 'inbox') DESC,    -- then the folder people watch
         last_synced_at ASC,              -- then least-recently-synced
         ...
```

Plus: **a run that is out of budget still owes one folder from the rotation.** A busy inbox otherwise eats the whole budget every time and the rotation never gets a turn.

- The symptom in your logs is *a constant folder count run after run*. Log the count; it is the only way you find this.
- Sort remote-store work by "longest un-visited" whenever a budget can cut the list short. Any fixed priority order starves its tail.

## The renumber trap

Remote ids are often only valid within a generation (IMAP `UIDVALIDITY`; equivalents exist wherever ids are per-collection sequences). When the generation changes, **every id you stored is meaningless** — and it does not mean the data is gone.

- On a generation change: move `uid` → `stale_uid`, set `uid = NULL`, reset the cursor, and **re-adopt rows by a stable identifier** (`Message-ID`) as they come back. Do not delete and re-download; you lose local state (reads, links, assignments) attached to those rows.
- **The local id column must be nullable, on purpose.** NULL is a real state: a draft not yet on the server, a message mid-move, a row awaiting re-adoption.
- **Ops holding an id from a previous generation must be deferred, not run.** Every op row carries the generation its id belongs to; if the server's differs, put the op back and let the sync re-adopt the row first. Running it deletes or flags a stranger.

## The write-back queue

- **Claim with `FOR UPDATE SKIP LOCKED`**, bump attempts, run, mark done. Bounded attempts (5) with growing backoff (30s → 2m → 10m → 30m → 1h), then a terminal `failed` the UI actually shows: *"N changes could not reach the server"* with a retry. A queue whose failures are invisible is a queue that silently loses writes.
- **The op payload must carry everything the op needs at run time**, because the source may be gone by then. Appending a sent copy needs the raw composed bytes in the payload — rebuilt later, it loses the attachments, whose bytes were freed when the send finished.
- **Every filter must know every op kind.** A real bug: a new `append_sent` op was created and called retryable, but the claim query excluded that kind and a staleness sweep treated its (correctly) NULL uid as dead work. The mail sent, the copy never filed, the op sat pending and nobody ran it. When you add a kind, grep every query that names kinds.
- **Batch compatible ops** — same folder, same flag change — into one command.
- **Some libraries report refusal by return value, not by throwing.** imapflow's `messageMove`/`messageDelete`/`messageFlagsAdd` return `false` on a NO/BAD response. Treat every `false` as a retryable failure; a codebase that only catches exceptions marks these done.

## Errors must carry the server's own words

Protocol libraries put the useless half in `message` and the actionable half on a side property: imapflow says `"Command failed"` and keeps `[AUTHENTICATIONFAILED] Invalid credentials` on `responseText`; nodemailer says `"Invalid login"` and keeps the `535` line on `response`.

Write one `errorText(e)` helper that keeps the message and appends whatever the library attached — reply, status, code, rejected command — skipping anything already contained in the message, capped in length. Store *that* on the account, the op, and the log line. `Last sync failed: Command failed` is not something an admin can act on; the same failure with the credentials line is.

## Say plainly when the mirror is stale

The mirror standing still **is the news** — the data on screen is older than it looks — and it must dominate the UI, not sit as a footnote next to a cheerful "last synced 3 minutes ago". Turn the status strip red with the error at the front.

- **Scope a user-triggered sync to what the user is looking at.** Someone reading one folder is asking about that folder; making them wait on the whole mailbox is the wrong trade. Offer a separate deliberate "sync everything" for the person who would rather wait.
- **A switched-off half is not a failure.** If receiving and sending are separate switches, a disabled half must report *why it was skipped* (`off` vs `not configured`) and the overall verdict must count only the halves actually tried — otherwise a deliberately read-only mailbox reads as broken.
- **Bound the first backfill** (e.g. last 90 days per folder) with an explicit "load older" that extends the window. Unbounded first sync is how you spend a day on a mailbox nobody wanted all of.
- **Suppress "new item" events during a folder's first backfill.** Announcing every message of a ten-year mailbox fans out thousands of webhooks and notifications for mail from 2019. Gate on `!firstSync`.

## Build the test seam before the engine

Define a narrow interface for everything the sync does to the server (`open`, `search`, `fetchFlags`, `download`, `move`, `append`, …) and have the engine talk only to that. The protocol client is one implementation; **an in-memory fake is the other, and it drives the entire engine end to end** — no mocking of the protocol library, ever.

The payoff is that you can write the *failure* into the fake. The fairness bug got a real regression test only because the fake could be told to **drop its connection after one folder per run**: with the old order, three folders are never reached; with the new one, every folder comes round.

- Test against a real server too, at least once (a throwaway Dovecot container), behind an env var so it skips itself in CI.
- **Fire-and-forget side effects need waiting on.** A test that read webhook delivery counts the instant the causing call returned failed at random, because the delivery lands a tick later. Poll for the count.

## Protocol traps (IMAP / imapflow, all verified in production)

- `changedSince` requires `client.enabled.has('CONDSTORE')` — the *capabilities* list is not the same thing as what got enabled.
- `fetch({ bodyParts })` returns content still transfer-encoded; use the download API instead.
- `download().meta.expectedSize` is the size of the whole message, not the part.
- Gmail files sent mail automatically — appending your own copy creates a duplicate. Skip the append when `X-GM-EXT-1` is advertised.
- **No `'error'` listener on the client crashes the process.** Attach one before anything else.
- Elect special folders (Sent, Drafts, Trash) by the server's `special-use` flags, not by name matching. Names are localized.

## Rendering untrusted remote HTML

Sanitize server-side *and* render inside a sandboxed iframe **without** `allow-scripts`, with its own CSP meta inside the frame. Park remote image URLs in `data-src` until the reader clicks "Show images" — remote images are read receipts.

## Checklist

- [ ] UI reads only the mirror; UI writes create a local row plus an op row and return.
- [ ] Ops drain before folders are read in the same run.
- [ ] One lease per account; cron and manual sync go through the same path.
- [ ] Wall-clock budget, per-folder cursor checkpoints, resumable backfill.
- [ ] Folder order: never-synced → watched folder → least-recently-synced; a starved run still owes one rotation folder.
- [ ] Folder count per run is logged, and someone would notice it never changing.
- [ ] Generation (UIDVALIDITY) checked per folder; reset re-adopts by stable id; local id column nullable.
- [ ] Ops carry their generation and defer rather than run against a renumbered folder.
- [ ] Every op-kind filter updated when a kind is added.
- [ ] `false` return values from the protocol client treated as retryable failures.
- [ ] Error text carries the server's reply, not just the library's summary.
- [ ] Failed sync is loud in the UI; "off" is distinguishable from "broken".
- [ ] First backfill bounded, and new-item events suppressed during it.
- [ ] A server interface with an in-memory fake that can be told to fail, plus one real-server test behind an env var.
- [ ] Remote HTML sanitized, framed without scripts, remote images opt-in.
