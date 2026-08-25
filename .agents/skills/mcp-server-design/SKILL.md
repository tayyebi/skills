---
name: mcp-server-design
description: Use when giving AI assistants (MCP clients) direct tool access to a product — per-user tokens that act as their owner, layered guards, a call log the owner reads, and a "wish" feedback tool so connected models report capability gaps instead of silently working around them.
---

# MCP Server Design (product tool access for AI assistants)

An MCP (Model Context Protocol) server built into a product lets a user connect their own AI assistant to their own data. The hard parts are not the protocol — a tools-only server is four JSON-RPC methods — but the authority model: whose power does the connected model wield, what stops it, and how do the humans find out what it did and what it wanted.

## When to use

Any time a product grows an MCP endpoint (or any "AI assistant can call our tools" surface). Complements `api-tokens` (the credential mechanics) — this skill is about what makes a MODEL-facing surface different from a REST API.

## The wish tool — teach models to file feedback, not work around gaps

The highest-leverage pattern here, and the cheapest. A connected model that lacks the right tool does not complain — it silently chains what exists (thirteen `list_candidates` calls standing in for one `count_candidates`) and the operators never learn a better tool was wanted. So give every connection a `wish` tool:

- **Input:** `wish` (the tool it wanted and what it should do, required) + `workaround` (what it did instead, optional — the workaround is often the best argument for building the tool).
- **Teach it up front.** The tool's own description says *when* to reach for it ("whenever the available tools do not quite fit — you had to chain calls, over-fetch, or give up"), and the server's `initialize` instructions repeat it. Mention it again in the "unknown tool" denial message — that is the exact moment a model is missing something.
- **Guard-exempt by design.** No scope, no role check, allowed on read-only tokens — the narrow, locked-down connections are precisely the ones most likely to hit a gap. Keep the rate limit: feedback is not an invitation to spam.
- **Own table, not the call log.** The call log is rolling traffic with truncated args; a wish is product feedback whose full text is the point. `mcp_wishes(id, tenant_id, token_id, token_name, user_id, wish, workaround, created_at)`.
- **A review surface for operators** (e.g. System → Wishes): newest first, attributed to tenant + token, with a dismiss-deletes action. An inbox nobody can empty stops being read.
- **Answer the model usefully:** confirm it was recorded for the humans, and tell it to finish the task with the tools that exist today — a wish is not an excuse to stop.

## Authority model

- **Tokens are personal and act as their owner.** No role stored on the token; resolve the owner's live role, active flag and tenant membership on **every** call. A demotion or deactivation then kills every token the user ever minted with nothing to revoke by hand.
- **Separate token family from the REST API** (distinct prefix, e.g. `app_mcp_` vs `app_live_`, separate table). An MCP token carries scopes and guards the REST verifier knows nothing about; sharing a table would let a "read-only assistant" credential authenticate against the full REST surface.
- **Bearer-auth only, never session cookies** — it is a machine surface, and a cookie-authenticated one is reachable by any page in the user's browser.
- **Bind the token to the tenant in the URL** and log the mismatch attempt against the token that made it.

## Guards, in a fixed order

One function decides what a call may do, cheapest and most absolute checks first:

```
owner-alive/expiry (at verify) → role → scope granted → read-only lock → rate limit
```

- **Scopes** come from one catalog that drives BOTH the mint-time UI checklist and enforcement — a scope the UI offers but the server never checks is a hole.
- **Read-only master lock** overrides every granted write scope, so a token can be made harmless without editing its scope list.
- **Optional per-token knobs:** hourly call cap, per-call row cap (trim + tell the model it was trimmed, rather than fail), an IP allowlist (exact/CIDR, empty = unrestricted, fail closed when a restricted token's caller IP can't be resolved), and an owner-written **safety prompt** served as the MCP `instructions` and repeated verbatim in every denial — the moment a model reaches for something it may not have is the moment the reminder is worth spending words on.
- **Denied ≠ error.** A guard refusal is an `isError` tool *result* with a machine-readable reason (the model can read it and adjust); an exception is an error. Log them as distinct outcomes — a denial means the guards worked.

## Everything is recorded

Per-call log the token's OWNER reads (not just admins): method, tool, outcome (`ok`/`denied`/`error`), duration, an args *summary* (never the full payload — free text becomes `<200 chars>`), row count, IP, user agent. This is also the UX for other features: the IP a call presents is the IP the owner copies into the allowlist.

## Pitfalls

- **Tool descriptions are for the model, not the UI** — keep them English and rich in "when to use" guidance; localize only the human-facing scope labels.
- Vocabulary the human uses (column names, stages) must reach the model **in the owner's language** at initialize, or the model guesses and moves the wrong record.
- **No delete tools**, in any configuration. An agent that destroys records on a misread instruction is not worth the convenience.
- Internal errors must not echo to the model — a raw message can carry SQL or another tenant's data into a context window. Log fully server-side; send a generic pointer.
- In-process rate limiting silently multiplies by replica count — fine on one box, wrong the day you scale; write that down where the limiter lives.
- Don't advertise tools the token can't call: filter `tools/list` through the same guard function (without consuming rate budget).

## Checklist

- [ ] `wish` feedback tool: guard-exempt, rate-limited, own table, operator review page, taught at initialize + in unknown-tool denials
- [ ] Distinct token prefix + table from the REST API; hash at rest, plaintext shown once
- [ ] Owner's live role/membership resolved per call (no role on the token)
- [ ] One scope catalog driving mint UI and enforcement; read-only master lock
- [ ] Guards in fixed order returning machine-readable deny reasons; denied logged apart from errored
- [ ] Row caps trim-and-tell; call log readable by the token owner
- [ ] Safety prompt in `instructions` and repeated on every denial
