---
name: content-addressed-file-storage
description: Use when an app stores user file uploads — a content-addressed blob store keyed by SHA-256 with automatic deduplication and reference counting, plus safe serving. Covers the node/blob split and orphan cleanup.
---

# Content-Addressed File Storage

Store each unique byte-stream once, keyed by its hash, and reference it from many places. Uploading the same file twice costs zero extra bytes.

## When to use

Any app with user uploads (attachments, resumes, images, documents) — especially multi-tenant, where the same file often appears many times.

## Core pattern

Split **content** from **naming**:

- **Blob** = the bytes, stored at a path derived from its digest: `data/<tenant>/<sha256>`. The hash *is* the identity. Table `blobs(sha256, size, mime, refcount, created_at)`.
- **Node** = a named reference (a "file" in a folder, an attachment on a record): `nodes(id, tenant_id, name, parent_id, blob_sha, owner, created_at)`. Many nodes can point at one blob.

Upload flow:
1. Stream upload to a temp file, computing SHA-256 as you go.
2. If the blob exists, increment `refcount`; else move the temp file into place and insert the blob row.
3. Create the node pointing at the blob.

Delete flow: remove the node, **decrement `refcount`**; when it hits zero, delete the blob file. (Or defer physical deletion to a periodic sweep for safety.)

## Serving safely

- Serve via a handler that checks authorization and streams the blob — **never** map user-supplied paths to the filesystem directly (path traversal).
- Set correct `Content-Type` and `Content-Disposition`; consider `X-Sendfile`/`X-Accel-Redirect` to let the web server stream large files.
- For sharing, issue expiring signed links rather than exposing blob paths (see `share-links-tracking`).

## Pitfalls

- **Refcount races** — increment/decrement under a transaction or the blob gets deleted while another node still needs it. Prefer deferred/swept deletion over immediate.
- **Tenant isolation** — hash-addressing dedups *within* a trust boundary; think carefully before deduping across tenants (a shared blob can leak existence/size info). Namespacing by tenant is the safe default.
- Path traversal / arbitrary file read in the serve handler.
- Storing huge files in the DB — store bytes on disk/object storage, metadata in the DB.
- Uploads that fill the temp dir — enforce size limits and clean temp files on failure.

## Checklist

- [ ] SHA-256 content addressing; blob table with refcount
- [ ] Node/blob split (many names → one blob)
- [ ] Refcount inc/dec transactional; zero-ref cleanup (swept)
- [ ] Authorized streaming serve handler, no path mapping
- [ ] Per-tenant namespacing; size limits; temp cleanup
