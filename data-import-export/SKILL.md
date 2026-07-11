---
name: data-import-export
description: Use when users need to get data in and out — CSV/tabular export and import with validation, error reporting, and large-file streaming. Covers encoding pitfalls, dry-run imports, and idempotent upserts.
---

# Data Import & Export

Let users bring data in (migration, bulk entry) and take it out (reporting, backup, leaving). A near-universal request; get the edge cases right once.

## When to use

Any app holding lists of records users maintain. Export is table stakes; import is what wins migrations from a competitor.

## Export

- **Stream**, don't buffer — for large datasets, write rows to the response as you page the query, so memory stays flat.
- **CSV correctness**: quote fields containing commas/quotes/newlines; escape quotes by doubling. Use a real CSV writer, not string concatenation.
- **Encoding**: emit UTF-8, and prepend a **BOM** if the audience opens files in Excel (otherwise non-ASCII mangles).
- **Formula injection**: prefix fields starting with `= + - @` with a `'` so spreadsheets don't execute them as formulas.
- Offer column selection and respect the requester's permissions (don't export fields they can't read).

## Import

- **Parse tolerantly, validate strictly.** Accept messy input (varied headers, extra whitespace, locale digits), then validate each row against a schema.
- **Dry-run first**: parse and validate the whole file, report row-by-row errors, and let the user confirm before any writes.
- **Map columns** — let the user match their headers to your fields; don't hard-require an exact template.
- **Idempotent upserts**: key on a natural id (email, external id) so re-importing updates rather than duplicating. Detect duplicates *within* the file too.
- **Partial success**: import valid rows, report failed ones with line numbers and reasons; don't fail the whole file for one bad row (unless the user asked for all-or-nothing).
- Run large imports as a **background job** with progress.

## Pitfalls

- Excel/CSV encoding (BOM, delimiters, quoted newlines) — the classic time sink.
- CSV formula injection — a security issue, not cosmetic.
- Duplicate creation on re-import — always upsert on a stable key.
- Loading the whole file/dataset into memory — stream both directions.
- Silent row drops — report exactly which rows failed and why.
- Locale-specific numbers/dates (`1.234,56`, Persian/Arabic digits) — normalize on parse.

## Checklist

- [ ] Streaming export; proper CSV quoting + UTF-8 BOM
- [ ] Formula-injection escaping on export
- [ ] Import: tolerant parse, strict validate, column mapping
- [ ] Dry-run with per-row error report; partial success
- [ ] Idempotent upsert on natural key; in-file dedup
- [ ] Large imports as background jobs with progress
