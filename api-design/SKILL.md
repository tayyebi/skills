---
name: api-design
description: Use when designing an HTTP API's resource surface — naming, verb semantics, status codes, and the request/response/error envelope shape. Covers what makes a REST API predictable to consume. Defers to versioned-http-api for versioning/OpenAPI, search-filtering-pagination for pagination, rate-limiting-abuse-prevention for throttling, and authorization-rbac/api-tokens for auth.
---

# API Design

Make the surface predictable: a consumer who has used one endpoint should be able to guess the shape of the next one without reading docs.

## When to use

Any HTTP API with more than one or two endpoints — internal service-to-service, or public. The cost of an inconsistent surface compounds: every inconsistency is a special case every client has to hardcode.

## Resource naming

- Collections are plural nouns: `/orders`, `/users/{id}/orders`.
- Pick one word-separator convention (kebab-case or snake_case) for multi-word resources and use it everywhere: `/order-items`, not a mix of `/order-items` and `/orderItems`.
- Nest one level deep at most: `/users/{id}/orders` is fine; `/users/{id}/orders/{id}/items/{id}` forces the client to carry three IDs to reach a resource that has its own ID. Prefer `/order-items/{id}` with the parent as a filter or field.
- Non-CRUD actions are verbs on a sub-path, not new HTTP semantics: `POST /orders/{id}/cancel`, `POST /password-resets/request` — not `PATCH /orders/{id}?action=cancel`.

## HTTP method semantics

| Method | Meaning | Idempotent | Has body |
|---|---|---|---|
| GET | Retrieve, no side effects | Yes | No |
| POST | Create, or trigger a non-idempotent action | No | Yes |
| PUT | Full replacement of a resource | Yes | Yes |
| PATCH | Partial update | No* | Yes |
| DELETE | Remove | Yes | No |

*A well-designed PATCH (e.g. JSON Merge Patch semantics) can be made idempotent; don't assume it is by default.

- Use PUT when the client sends the complete resource state; use PATCH when it sends only the fields that changed.
- Route complex queries that don't fit a query string to `POST /orders/search` with a body — still semantically a read, so don't call it "create."
- If retries matter for a POST (payment, order placement), see `idempotency-keys` — a client-supplied key, not the HTTP method, is what makes POST safely retryable.

## Status codes

Use the specific code, not the nearest-enough one — clients branch on it.

```
200 OK              success, body returned
201 Created         resource created; include Location header
202 Accepted         request queued for async processing
204 No Content       success, no body (typical for DELETE)
400 Bad Request      malformed request (unparseable body, wrong type)
401 Unauthorized     missing or invalid credentials
403 Forbidden        valid credentials, insufficient permission
404 Not Found        resource does not exist (or caller can't know it does)
409 Conflict         concurrent edit, duplicate, state-transition violation
422 Unprocessable    well-formed request, semantically invalid (failed business rule)
429 Too Many Requests rate limit exceeded
500 Internal Error   unexpected server fault — never expected/routine
```

400 vs 422 is the distinction worth holding onto: 400 means "I couldn't even parse this," 422 means "I understood it and it's invalid" (missing required field, value out of range, referenced entity doesn't exist). Field-level validation errors are almost always 422.

## Response envelope

Wrap every response in a consistent top-level shape so clients write one parser, not one per endpoint.

```
// Single resource
{
  "data": {
    "type": "order",
    "id": "ord_abc123",
    "attributes": { "status": "pending", "total": "29.99" }
  }
}

// Collection — pagination fields per search-filtering-pagination
{
  "data": [ { "type": "order", "id": "...", "attributes": {...} } ],
  "meta": { "total": 152 },
  "links": { "next": "...", "prev": null }
}
```

- Give every entity an explicit `type` and stable string `id` — cast numeric IDs to strings so JSON-number precision limits in some client languages can't silently corrupt large IDs.
- Return only fields the consumer needs; an explicit allow-list beats "serialize the internal record and hope nothing sensitive is on it."

## Error envelope

One shape for every error, everywhere in the API — never let a framework's default error page leak through with a different shape than your handwritten ones.

```
{
  "errors": [
    {
      "code": "INSUFFICIENT_BALANCE",
      "message": "Insufficient account balance",
      "field": "amount"
    }
  ]
}
```

- `code` is a stable, machine-matchable string (client code branches on this) — never renumber or repurpose one once shipped.
- `message` is for humans/logs, not for branching logic; it can change wording without breaking clients.
- `field` (or a pointer into the request body) is present for validation errors and absent otherwise.
- Return *all* field errors from one validation pass, not just the first — a client that fixes one field only to be told about the next on the following request is a bad round trip.

## Pitfalls

- Reusing 200 for everything and encoding the real status inside the body (`{ "success": false }`) — status codes exist so infrastructure (proxies, clients, monitoring) can act on them without parsing the body.
- 404 vs 403 leaking existence: for a resource an unauthorized caller shouldn't even know exists, return 404, not 403.
- Inconsistent pluralization or casing across endpoints — pick the convention once, lint for it.
- Verbs that don't match method semantics: a `GET` that mutates state, a `DELETE` that isn't idempotent (returns an error on the second call for an already-deleted resource — it shouldn't).
- A different error shape per endpoint or per exception type — centralize error serialization in one place so it's structurally impossible to drift.

## Checklist

- [ ] Plural, one-level-deep, consistently-cased resource paths; actions as sub-path verbs
- [ ] HTTP method matches semantics (idempotency, body, side effects)
- [ ] Specific status codes used (400 vs 422 distinguished; 404 hides unauthorized existence)
- [ ] One response envelope shape (`data`/`meta`/`links`) across all endpoints
- [ ] One error envelope shape (`code`, `message`, optional `field`) across all endpoints
- [ ] IDs are stable strings; response fields are allow-listed, not raw internal records
- [ ] Versioning, pagination, rate limits, and auth deferred to their own skills
