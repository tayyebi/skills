---
name: backup-restore
description: Use when an app owns data users can't afford to lose — scheduled full backups with offsite copies, tested restore, and retention. Covers what to include (DB + blobs), encryption, and the "untested backup is not a backup" rule.
---

# Backup & Restore

Protect against data loss and enable disaster recovery — with backups that are actually restorable.

## When to use

Any self-hosted app holding customer data. Managed databases give you some of this; you still own application blobs, config, and the restore *procedure*.

## What to back up

A backup is only complete if it captures **everything needed to reconstitute the app**:

- **Database** — logical dump (`pg_dump`) or physical snapshot. Logical dumps are portable across versions; snapshots are faster to restore.
- **Blob/file storage** — the content-addressed uploads (see `content-addressed-file-storage`). A DB restore without its blobs gives you dangling references.
- **Config/secrets** — enough to bring the app up (documented separately if secrets shouldn't live in the backup).

Keep DB and blobs **consistent** — snapshot around the same point, or accept blob ≥ DB (extra unreferenced blobs are harmless; missing ones are not).

## Scheduling, offsite, retention

- **Schedule** frequent backups (e.g. every 15 min incremental / nightly full) via the job scheduler (see `background-jobs-scheduling`).
- **Offsite copy** — push to a *different* machine/provider (FTP/object storage/S3). A backup sitting on the same disk that dies with the server is not a backup.
- **Retention** with tiering: keep recent frequent + older sparse (GFS: grandfather-father-son). Prune automatically.
- **Encrypt** backups at rest and in transit; they contain all your sensitive data.

## Restore (the part people skip)

- Provide an explicit **restore** operation (with a scary confirmation) and a **download** of any backup.
- **Test restores regularly** — an untested backup is a hope, not a backup. Periodically restore into a scratch environment and verify row counts / a smoke test.
- Track backup runs (size, duration, status) and **alert on failure or staleness** — a silently-broken backup job is discovered at the worst possible time.

## Pitfalls

- DB backed up, blobs not (or vice versa) → inconsistent restore.
- Offsite copy missing → single point of failure.
- Never test-restoring → discovering the dump is corrupt during an actual incident.
- Unencrypted offsite backups → a breach of the backup store is a full data breach.
- No failure alerting → weeks of "successful" empty backups.

## Checklist

- [ ] Backs up DB **and** blobs (consistent point) **and** config
- [ ] Scheduled, with offsite copy to a different provider
- [ ] Encrypted at rest and in transit
- [ ] Retention/pruning (tiered)
- [ ] Restore + download operations; periodic tested restores
- [ ] Run history + failure/staleness alerts
