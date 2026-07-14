---
name: git-workflow-conventions
description: Use when writing commit messages, naming branches, opening PRs, or deciding rebase vs. merge — conventional-commit format, branch naming, PR size discipline, and trunk-based vs. feature-branch workflow, independent of language or stack.
---

# Git Workflow Conventions

A Git history is a product, read later via `git blame`/`git log`/`git bisect` by someone (often you) who wasn't in the room. Agreed-upon conventions make that history searchable and bisectable instead of an archaeology dig.

## When to use

Every repo with more than one contributor, from day one. Retrofitting conventions onto an inconsistent history is far more expensive than starting clean.

## Commit messages

Use **Conventional Commits**:

```
<type>(<scope>): <short summary>

<optional body — what and why, not how>

<optional footer — breaking changes, issue references>
```

| Type | When |
|------|------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes nor adds a feature |
| `perf` | Performance improvement |
| `test` | Adding or updating tests |
| `docs` | Documentation only |
| `style` | Formatting only, no code change |
| `chore` | Build, CI, tooling, dependencies |
| `ci` | CI/CD configuration |
| `revert` | Revert a previous commit |

```
feat(orders): add bulk cancellation endpoint
fix(payments): prevent double-charge on gateway timeout
refactor(auth): extract token generation into its own module
perf(products): add composite index on status+created_at
chore(deps): bump the HTTP client library to v2
```

- Summary line: imperative mood ("add", not "added"), ≤72 chars, no trailing period.
- Body: wrap ~72 chars, explain **what** and **why** — the diff already shows *how*.
- Breaking change: `!` after the type (`feat!(api): remove v1 endpoint`) or a `BREAKING CHANGE:` footer.

## Branch naming

```
<type>/<issue-ref>-<short-description>
```

| Prefix | Purpose |
|--------|---------|
| `feat/` | New features |
| `fix/` | Bug fixes |
| `refactor/` | Refactoring |
| `chore/` | Maintenance |
| `hotfix/` | Critical production fixes |

Examples: `feat/PROJ-123-bulk-order-cancel`, `fix/issue-456-double-charge`, `hotfix/1.2.1-payment-crash`.

## Pull requests

Title mirrors the commit-summary convention. Body: **Summary** (what changed), **Motivation** (why, with issue link), **Changes** (bulleted, what not how), **Testing** (checklist: unit/integration pass, manual QA steps), **Screenshots** for UI changes, and a checklist covering lint/style conformance, updated public-API docs, reversible schema migrations, and no leftover debug code.

- Keep PRs small — under ~400 lines changed is a good target. A reviewer should understand the whole change in under 30 minutes.
- Split large work into a stack of small, independently reviewable PRs rather than one sprawling one.

## Rebase vs. merge, and workflow shape

**Trunk-based (default recommendation)**: short-lived feature branches off `main`, merged within a day or two.
- Rebase the feature branch onto `main` before merging — never merge `main` into the feature branch. Rebasing keeps history linear; merging `main` in repeatedly buries the actual change under noise commits.
- Squash-merge single-purpose branches into one clean commit; use a real merge commit only for a multi-commit logical unit that was reviewed and is worth preserving commit-by-commit.
- Delete branches immediately after merge.

**Feature-branch / release-train workflow** (`main ← develop ← feat/foo`) is heavier but still correct when a product genuinely ships multiple supported release lines at once. Don't default to it out of habit — trunk-based is simpler and fits most teams.

## Git hooks

- **pre-commit**: lint/format staged files, run static analysis on the changed files only — fast, local feedback.
- **commit-msg**: reject commits that don't match the conventional-commit format before they land.
- **pre-push**: run the fast test suite so broken code never reaches the shared branch.

## .gitignore essentials

Build artifacts and dependency directories, `*.log`, `.env` and `.env.local` (commit a `.env.example` with placeholders instead), IDE directories (`.idea/`, `.vscode/`), OS files (`.DS_Store`, `Thumbs.db`), and test/coverage caches.

## Pitfalls

- Committing `.env` or any secret file — if it happens, rotate the secret immediately; deleting the commit doesn't remove it from history until a rewrite, and even then treat the value as compromised.
- Committing generated/build output that the deploy pipeline doesn't actually need in-repo.
- Amending or force-pushing commits already pushed to a shared branch — it rewrites history other people have built work on top of.
- Giant PRs that take a reviewer half a day to get through — split into a stack instead.
- Merging `main` into a feature branch repeatedly instead of rebasing — produces a tangled, non-linear history that's hard to bisect.

## Checklist

- [ ] Commit messages follow `type(scope): summary`
- [ ] Branch named `<type>/<ref>-<description>`
- [ ] PR under ~400 lines, or explicitly split into a stack
- [ ] Rebased onto the trunk before merge; squashed if single-purpose
- [ ] No secrets, generated output, or IDE files committed
- [ ] pre-commit / commit-msg / pre-push hooks enforce the above automatically
