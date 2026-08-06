# Errors and resilience

How the system behaves when something goes wrong. Load when the codebase calls other services, uses
queues, retries, or does long-running work — or whenever it has a `try`/`catch`.

## Contents

- Leaking internals in errors
- Swallowed errors
- Unhandled rejections and crashes
- Missing timeouts
- Retries done wrong
- Circuit breaking and degradation
- Resource leaks
- Graceful shutdown
- Health checks
- Background work on serverless
- Caching failure modes

---

## Leaking internals in errors

**Signal:**

```js
catch (err) { res.status(500).json({ error: err.message, stack: err.stack }); }
```
```python
DEBUG = True                      # in the production settings path
```

**Why it breaks:** Stack traces reveal file paths, framework versions, and internal structure. ORM
errors echo SQL and table names. Django's debug page shows environment variables — including secrets.
This is how an attacker maps your system before attacking it, and Django/Flask/Rails debug pages
turn a stack trace into a full compromise.

**Fix:** Log the full error server-side with a correlation id; return a generic message plus that id.

```js
catch (err) {
  const id = crypto.randomUUID();
  logger.error({ id, err });
  res.status(500).json({ error: { code: 'internal_error', message: 'Something went wrong.', id } });
}
```

Confirm `DEBUG`/`APP_DEBUG`/`NODE_ENV` is set correctly for production, and that the check is on an
environment variable rather than a hardcoded value someone forgot to flip. Verify there is a global
error handler so an unhandled error in any route still produces a clean response — many frameworks
default to a verbose page.

---

## Swallowed errors

**Signal:**

```js
try { await doThing(); } catch (e) {}
try { await doThing(); } catch (e) { console.log(e); }   // logged, then continues as if fine
```
```python
except Exception:
    pass
```

**Why it breaks:** The operation failed, the caller was told it succeeded, and nothing is logged
where anyone looks. Data loss with no signal. This is the single hardest class of bug to diagnose
later because there is no evidence it ever happened.

**Fix:** Every catch does one of three things, deliberately: recover with a documented fallback,
rethrow after adding context, or fail the request. Catching to log-and-continue is only correct when
the operation is genuinely optional, and that should be stated in a comment.

Never catch broad exception types around a block that does several things — you cannot know which one
failed. Catch narrowly, around the statement that can fail.

---

## Unhandled rejections and crashes

**Signal:** `async` route handlers in Express 4 without a wrapper. Promises created without `await`
or `.catch`. No `process.on('unhandledRejection')`. A `setInterval` callback that can throw.

**Why it breaks:** An unhandled rejection in an Express 4 async handler never reaches the error
middleware — the request hangs until the client times out. In newer Node an unhandled rejection
terminates the process, so one bad request restarts the server for everyone.

**Fix:** Express 4: wrap every async handler (`express-async-errors`, or a `wrap(fn)` helper). Express
5 handles it natively. Register process-level handlers for `unhandledRejection` and
`uncaughtException` that log and then exit — do not swallow and continue, the process state is
unknown. Let the supervisor restart it.

---

## Missing timeouts

**Signal:** `fetch(url)`, `axios.get(url)`, `requests.get(url)`, a database client, or an HTTP client
created with no timeout configured.

**Why it breaks:** Default timeouts are often infinite. One slow upstream holds every request thread
or connection until the whole service stops responding — including endpoints that do not touch that
upstream. This is the most common cause of a total outage caused by a partial failure.

**Fix:** An explicit timeout on every network call: HTTP client, database, cache, queue, and the
server's own request timeout. Make them shorter as you go deeper — if the gateway gives you 30s, the
upstream call gets 5s, so you can return a useful error instead of being killed.

```js
const res = await fetch(url, { signal: AbortSignal.timeout(5000) });
```
```python
requests.get(url, timeout=(3.05, 10))     # connect, read
```

Set connect and read timeouts separately. Set a query timeout on the database
(`statement_timeout`) so one bad query cannot pin a connection indefinitely.

---

## Retries done wrong

**Signal:** A fixed-delay retry loop. Retrying every error including 400s. Retries at several layers
at once (client retries, gateway retries, and the library retries).

**Why it breaks:** Retrying a 400 or a 403 will never succeed and just burns budget. Fixed-interval
retries from many clients synchronize into a thundering herd that keeps a recovering service down.
Layered retries multiply: 3 × 3 × 3 is 27 requests for one logical call.

**Fix:** Retry only what can succeed on a second attempt: timeouts, connection errors, 429, 502, 503,
504. Never retry 4xx other than 429 and 408. Use exponential backoff with jitter, a small cap (3
attempts), and honour `Retry-After`. Retry at exactly one layer. Every retried write needs
idempotency (`data-layer.md`) or the retry duplicates the effect.

---

## Circuit breaking and degradation

**Signal:** A non-essential dependency — analytics, recommendations, avatar service, email — whose
failure fails the whole request.

**Why it breaks:** Availability multiplies. Five dependencies at 99.9% each, all required, gives
99.5%. A checkout that fails because the analytics call failed is a self-inflicted outage.

**Fix:** Classify each dependency as required or optional. Optional ones get a short timeout, a
fallback value, and failure that is logged but not propagated. For repeated failures, stop calling
temporarily (circuit breaker) rather than waiting for the timeout on every request. Decide and
document what "degraded" looks like for each feature.

---

## Resource leaks

**Signal:** `open(` without a context manager or `finally`. A database connection acquired and
released only on the success path. Event listeners added per request. An unbounded in-memory cache,
array, or `Map` that only ever grows.

**Why it breaks:** Slow memory growth ending in an OOM kill hours or days after deploy, or file
descriptor exhaustion that makes every new connection fail. Hard to reproduce, easy to prevent.

**Fix:** `with`/`using`/`defer`/`try-finally` for anything acquired. Release in `finally`, not after
the last statement. Bound every in-memory structure with a size limit and eviction. Remove listeners
and clear timers when the owning object goes away. If memory grows monotonically under steady load,
that is a leak regardless of how slow it is.

---

## Graceful shutdown

**Signal:** No `SIGTERM` handler. The process exits immediately on deploy.

**Why it breaks:** Every deploy kills in-flight requests and mid-transaction work. Users see errors
on every release, and partially completed writes accumulate.

**Fix:** On `SIGTERM`: stop accepting new connections, finish in-flight requests up to a deadline,
stop pulling from queues, flush logs and metrics, close database pools, then exit. Make sure the
shutdown deadline is shorter than the platform's kill timeout. Add a short delay before closing so
the load balancer notices the instance is draining.

---

## Health checks

**Signal:** No health endpoint, or one that returns 200 unconditionally, or one that checks every
dependency.

**Why it breaks:** A health check that always returns 200 keeps routing traffic to a broken instance.
A health check that fails when an optional dependency is down takes the whole fleet out of rotation
for something that should have degraded gracefully.

**Fix:** Two endpoints. **Liveness**: is the process alive — cheap, no dependencies, restart if it
fails. **Readiness**: can it serve traffic — checks only the dependencies without which it cannot
function at all (usually just the database). Keep both fast and unauthenticated but uninformative;
do not expose versions or internal state.

---

## Background work on serverless

**Signal:** `doWorkLater()` called without `await` before returning a response, on Vercel, Lambda,
Cloud Functions, or Cloud Run.

**Why it breaks:** The runtime may freeze or terminate the instance once the response is sent. The
work silently does not happen — sometimes. It works locally and in testing, then fails
non-deterministically in production, which makes it very hard to trace.

**Fix:** Use the platform's mechanism for post-response work (`waitUntil`, a durable workflow, a
queue). For anything that must not be lost, put it on a queue with retries — not in-process.

---

## Caching failure modes

**Signal:** Cache keys without the user or tenant in them. Caching an authenticated response at a CDN.
No stampede protection. No handling of a cache outage.

**Why it breaks:** A cache key missing the user id serves one user's data to another — a data breach
caused by a performance optimization. `Cache-Control: public` on a personalized response does the
same at CDN scale. When a hot key expires, every concurrent request goes to the database at once.

**Fix:** Include every dimension that changes the response in the key: user or tenant, role, locale,
version. Mark authenticated responses `Cache-Control: private, no-store`. Use a lock or
stale-while-revalidate so only one request recomputes. Treat a cache failure as a miss, never as an
error — if the cache being down takes the app down, the cache is a dependency, not a cache. Set a TTL
on every key.
