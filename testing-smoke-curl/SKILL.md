---
name: testing-smoke-curl
description: Use when you need the smoke-test gate with zero dependencies beyond curl/jq/awk — no Python, no browser, copy-paste shell one-liners that assert status codes and payload shapes, tally pass/fail counters, and emit a machine-readable JSON summary for a CI gate. The shell-only counterpart to [[testing-smoke]]; for full API coverage see [[testing-api]]; for an RBAC matrix see [[testing-authz-matrix]]; for billing depth see [[testing-billing]].
---

# Testing Smoke Test (curl-only)

The zero-dependency version of a deployment smoke gate: every check is a `curl` call piped through `jq`, wrapped in a tiny `check()` helper, runnable from any shell with no test framework installed. Useful when the target environment (a CI runner, a bastion host) can't or shouldn't run a full test toolchain.

## When to use

- CI pipelines where installing a test framework is more overhead than the check itself.
- Any environment where you only have shell + curl + jq (a deploy box, an on-call laptop).
- As a literal copy-paste playbook a human can run by hand during an incident.

## Core pattern

A single helper makes every check self-describing and keeps a running tally:

```bash
BASE=https://your-target.example
PASS=0; FAIL=0
check() { [ "$1" = "$2" ] && { PASS=$((PASS+1)); echo "PASS: $3"; } \
                           || { FAIL=$((FAIL+1)); echo "FAIL: $3 (got $1 want $2)"; }; }

# Health
check "$(curl -so /dev/null -w '%{http_code}' "$BASE/api/health")" "200" "health"

# Auth round-trip
TOKEN=$(curl -sS -X POST "$BASE/api/auth/login" -H 'Content-Type: application/json' \
  -d '{"email":"probe@example.test","password":"..."}' | jq -r .token)
check "$(curl -so /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN" "$BASE/api/me")" "200" "auth"

# CRUD round-trip: create, then confirm it reads back
ID=$(curl -sS -X POST "$BASE/api/widgets" -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"name":"smoke"}' | jq -r .id)
check "$(curl -sS -H "Authorization: Bearer $TOKEN" "$BASE/api/widgets/$ID" | jq -r .name)" "smoke" "crud round-trip"

# Auth guard: unauthenticated request must be rejected, not accepted
check "$(curl -so /dev/null -w '%{http_code}' "$BASE/api/me")" "401" "auth guard"
```

Structure the full script in tiers (up → auth → reference data → write path → secondary domain), same as [[testing-smoke]] — the difference here is purely mechanical: no framework, just `curl`, `jq`, and shell conditionals.

## Design decisions

- **One `check()` helper, used everywhere.** Every assertion becomes a one-liner: run a curl, compare to expected, print PASS/FAIL, increment a counter. Consistency here is what makes the output scannable.
- **Assert status codes AND payload shape.** A 200 with an empty/wrong body is still a failure — check `jq` output, not just `%{http_code}`, wherever the response body carries the signal (token length, a specific field value, an array length).
- **Emit a machine-readable summary at the end**, not just human-readable PASS/FAIL lines, so CI can gate on it without scraping text:

```bash
jq -nc --arg gate "$([ "$FAIL" -eq 0 ] && echo pass || echo fail)" \
  --argjson pass "$PASS" --argjson fail "$FAIL" \
  '{gate:$gate, totals:{pass:$pass, fail:$fail}}'
[ "$FAIL" -eq 0 ] || exit 1
```

- **Known, currently-accepted issues get a distinct status**, not a silent pass or a blocking fail — track them (e.g. a `KNOWN` bucket) so a pre-existing bug doesn't red the whole gate, but also doesn't quietly disappear from the report.
- **Clean up at the end of the script** — delete anything created (test records, sessions) so re-runs don't accumulate garbage.

## Pitfalls

- **Checking only the HTTP status** when the bug is in the body — a 200 with `{"error": "..."}` still needs a `jq` assertion on the payload.
- **No CI-parseable output** — a wall of colored PASS/FAIL text is fine for a human, useless for a pipeline; always emit a final JSON summary and a real exit code.
- **Hardcoding credentials in the script** instead of environment variables — breaks portability and risks a leaked secret in shell history/CI logs.
- **Silent variable failures** — `jq -r .token` on an error response returns `null` as a string; assert length/format, don't just assume success.
- **No cleanup step** — repeated smoke runs against a shared environment leave a trail of throwaway records.

## Checklist

- [ ] `check()` helper wraps every assertion; consistent PASS/FAIL output
- [ ] Status codes AND response payload shape both asserted
- [ ] Auth guard checked (unauthenticated request rejected)
- [ ] CRUD or core write path round-trips end-to-end
- [ ] Known/accepted issues tracked in their own bucket, not silently passing or failing the gate
- [ ] Machine-readable JSON summary emitted; nonzero exit on any hard failure
- [ ] Script cleans up anything it created
