---
name: spf-record-validation
description: Use when sending mail from a domain you control — resolving an SPF record the way a receiver does, counting it against the RFC 7208 ten-lookup ceiling, and monitoring it, because the count is spent across records you do not own and breaks without you changing anything.
---

# SPF Record Validation

SPF looks like a DNS record you set once. It is really a **recursive program executed by every receiver**, with a hard instruction budget, whose body is mostly written by other people. That is why it deserves a tool and a check, not a one-time paste into a DNS panel.

## The fact that drives everything

RFC 7208 §4.6.4 allows a receiver **ten DNS-querying mechanisms** per evaluation. Over that, the receiver returns `permerror` and your mail stops being authorised.

Those ten are spent across records you do not control. `include:_spf.provider.example` might cost one lookup today and four next month because the provider added a sub-include three levels down. **Nobody at your end touched anything and SPF broke.** A validator you run once tells you nothing about that; a scheduled check tells you before the mail stops.

## What costs a lookup

| Costs one | Free |
|---|---|
| `include:` | `ip4:` / `ip6:` |
| `a` / `mx` / `ptr` / `exists` | `all` |
| `redirect=` | `exp=` (not evaluated during the check) |

Also capped: `mx` may expand to at most **10 address records**, and more than **2 void lookups** is itself a `permerror`.

## Resolving it the way a receiver does

Walk the record and follow every `include:` and `redirect=`, resolving `a` and `mx` down to addresses.

**Keep walking past ten, up to a hard cap.** Stopping at the limit lets you report "at least 10", which is not actionable. Walking to a cap (~40 mechanisms, ~12 levels) lets you report **"14 of 10"** — the operator now knows whether they need to remove one include or restructure the whole record.

**Break cycles on the include chain, not a global visited set.** A domain legitimately reached down two different branches is counted twice, because that is what receivers do. A global "already seen" set under-counts and tells you the record is fine when a receiver will reject it.

**Count mechanisms, not queries.** If you cache DNS answers within a run (you should), the query count and the RFC count diverge. Count the mechanisms as you walk them; let caching only make the walk faster.

## Findings worth reporting

- **Over the ten-lookup limit** — `permerror`, mail unauthorised.
- **Exactly at ten** — one more include anywhere in the chain, by anyone, breaks it. Worth a warning of its own.
- **Void lookups over two** — also `permerror`. A void lookup is an *empty* answer (NXDOMAIN, or no records of that type), which is a different thing from a lookup that *failed*. Conflating them is the most common bug in a homemade checker.
- **More than one SPF record published** — `permerror`. Receivers do not pick one.
- **`+all`** — authorises the entire internet to send as the domain.
- **No `all` and no `redirect=`** — receivers fall back to `neutral`, which is close to having no policy.
- **`ptr`** — deprecated (§5.5), slow and unreliable at receivers.
- **`mx` expanding past 10 hosts** — over the address-record cap.
- **Unknown mechanism or unparseable term** — `permerror`.
- **Unknown modifier** — ignored by receivers, so usually a typo doing nothing.

### Two that are easy to report wrongly

**A record that is only `redirect=` has no `all` of its own.** `gmail.com` is one. Reporting "no policy" for it is wrong in the alarming direction — resolve the redirect target and report the **inherited** policy, noting where it came from.

**`redirect=` alongside `all` is dead weight**, not a conflict. §6.1: the `all` decides and the redirect is never evaluated. Say "ignored", not "broken".

**Macros** (`%{i}`, `%{d}`…) expand per message. They still cost a lookup — charge it — but there is nothing to resolve without an actual message in hand. Say so rather than reporting a failure.

## Flattening to addresses

Collect every authorised `ip4`/`ip6` and resolved address into a flat list. It is the thing operators actually paste elsewhere — into a firewall, an allow-list, or a diff against last week's.

**Keep the prefix length.** `/24` is the difference between one host and 256, and stripping it silently widens or narrows whatever you paste it into.

Flattening the *published record* is a different matter: it removes lookups by inlining a provider's current addresses, and you inherit responsibility for every future change they make with no notice. Treat it as a last resort with a monitoring obligation attached, not as the fix.

## Resolving DNS for this

You are walking records on nameservers you do not control, from a request path.

- **Two budgets, not one**: a query count *and* a wall clock. A chain pointing at a black-holed nameserver otherwise holds the request open for minutes.
- **DoH first, plain DNS as fallback** (or the reverse) — and use the *same* resolver configuration as any other checks in the system, so both halves see the same answers and don't produce contradictory reports.
- **An empty answer is not an exception.** A name that doesn't exist, or exists without records of the requested type, is an empty list. Only a `SERVFAIL`/timeout/refusal is an error. The whole void-lookup concept depends on this distinction.
- **TXT character-strings concatenate with nothing between them** (§3.3). A record longer than 255 bytes is published as multiple strings and must be joined with no separator — a space between them silently corrupts the record you analyse.
- Retry over TCP when a UDP answer comes back truncated.

## Monitoring, not just validating

Run it on a schedule and alert on the transitions, because the failure arrives without a local change:

- lookup count crossing 10 (and crossing 9, as early warning)
- void lookups crossing 2
- the effective policy qualifier changing (`-all` → `~all` → `?all` → `+all`)
- the flattened address set changing — a provider rotating addresses is normal, a set that empties is not
- the record disappearing or becoming two records

## Pitfalls

- **Treating NXDOMAIN as a failure** — it is a void lookup, which counts toward a different limit.
- **Counting DNS queries instead of mechanisms** — caching makes them differ, and the RFC counts mechanisms.
- **Stopping the walk at the limit** — "at least 10" is not actionable; "14 of 10" is.
- **A global visited set for loop-breaking** — under-counts, and reports a broken record as healthy.
- **Reporting "no policy" for a redirect-only record** — alarming and wrong.
- **Joining multi-string TXT records with a space** — corrupts long records.
- **Checking only the top-level record** — the whole point is what the includes cost.
- **Assuming your sending IP being listed means SPF passes** — over the lookup limit, `permerror` beats the match.
- **Flattening as a fix** — you have adopted the provider's IP churn as your own maintenance burden.

## Checklist

- [ ] Follows `include:` and `redirect=`; resolves `a` and `mx` to addresses
- [ ] Counts DNS-querying mechanisms against 10; keeps walking to a hard cap
- [ ] Void lookups counted separately from failed lookups, against 2
- [ ] Cycles broken on the chain, so legitimate double-counting is preserved
- [ ] Reports duplicates, `+all`, missing `all`, `ptr`, `mx` over 10, unparseable terms
- [ ] Redirect-only records report the inherited policy, not "none"
- [ ] `redirect=` with `all` reported as ignored, not as an error
- [ ] Macros charged a lookup and reported as unresolvable rather than failed
- [ ] Flat address list keeps prefix lengths
- [ ] Query budget *and* wall clock; multi-string TXT joined with no separator
- [ ] Scheduled, with alerts on the count and the policy qualifier changing
