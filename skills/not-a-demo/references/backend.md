# Backend and data

## Queries

- Paginate every list endpoint: default 20-50 items, a hard maximum (e.g. 100). Cursor-based for feeds and large or fast-changing tables; offset is fine for small admin lists.
- No queries inside loops (N+1). Use joins, eager loading (`include`, `select_related`, `with`) or a DataLoader. A page's query count must not grow with its row count.
- Index the columns used in `WHERE`, `JOIN` and `ORDER BY` on large tables, including foreign keys and the columns RLS policies filter on. Confirm with `EXPLAIN`. Don't index everything; every index slows writes.
- One connection pool per process, created once. On serverless, use a pooler (PgBouncer or the provider's pooled URL).

## Work outside the request

- Anything over about 1s, or that can fail on its own, goes to a background job: PDF exports, emails, imports, image processing, webhook handling. Return a job ID; the UI shows progress and tells the user when it's done.
- Every outbound call has a timeout (5-10s). Retry only idempotent calls, with exponential backoff and jitter. If it still fails, degrade instead of failing the whole request.
- Independent calls run in parallel (`Promise.all`, `asyncio.gather`).

## Correctness

- Multi-step writes run in a transaction; a failure rolls everything back.
- Idempotency keys on POSTs that charge money or have side effects ([saas-essentials.md](saas-essentials.md)).
- Money as integers in minor units plus a currency code. Never floats.
- Instants in UTC; future local events as local time + IANA zone.
- Migrations that don't lock tables ([saas-essentials.md](saas-essentials.md)).

## Responses

- One error shape everywhere, e.g. `{ "error": { "code", "message", "fields" } }`, with the right status: 400 validation, 401 not signed in, 403 not allowed, 404 not found or not yours, 409 conflict, 429 rate-limited, 500 generic. Field errors map onto form fields.
- Return only the fields the client needs.
- Brotli or gzip: check the host or CDN first; most already compress.

## Caching

- Start with HTTP caching headers and in-process memoization. Add Redis only with several instances or a measured hot path. Decide how each cached value gets invalidated before adding it.
- Prefer the framework's built-ins (Rails Solid Cache and Solid Queue, Next.js caching, Django's cache) before new infrastructure.

## Logging

- One structured log line per request: request ID, user ID, route, status, duration, DB time, query count. Pass the request ID on to jobs and outbound calls.
- Never log secrets, tokens or passwords ([security.md](security.md)).

## Dependencies

- Boring tech by default: Postgres or SQLite and the framework's own auth, jobs and cache.
- Remove unused dependencies (`knip`, `depcheck` or the language's equivalent): smaller installs, faster cold starts, fewer security alerts.
