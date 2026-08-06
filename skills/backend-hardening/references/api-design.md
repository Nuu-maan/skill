# API design and contracts

The shape of what you expose. Load when the codebase has HTTP endpoints consumed by a frontend or a
third party.

## Contents

- Status codes
- Response shape and error format
- CORS
- Pagination
- Versioning and breaking changes
- HTTP method semantics
- Security headers
- Content type and body limits
- Over-fetching in responses
- Long-running work
- Documentation

---

## Status codes

**Signal:** `res.status(200).json({ success: false, error: ... })`. Everything returning 200. A
`catch` block that returns 500 for validation failures.

**Why it breaks:** Clients, proxies, CDNs, monitoring, and retry logic all key off the status code. A
200 with an error body means failures are invisible in metrics, retried incorrectly, and cached.

**Fix:** Use the code that describes what happened.

| Code | Meaning |
| :--- | :--- |
| 200 / 201 / 204 | Success / created / success with no body |
| 400 | Malformed or invalid input |
| 401 | Not authenticated — no or bad credentials |
| 403 | Authenticated but not permitted |
| 404 | Does not exist, **or exists but the caller may not know that** |
| 409 | Conflict — duplicate, version mismatch |
| 422 | Well-formed but semantically invalid, where you distinguish it from 400 |
| 429 | Rate limited (include `Retry-After`) |
| 500 | Your bug |
| 502 / 503 / 504 | Upstream failed / unavailable / timed out |

401 vs 403 is a real distinction: 401 means "log in", 403 means "logging in again will not help".
Prefer 404 over 403 for objects whose existence is sensitive.

---

## Response shape and error format

**Signal:** Every endpoint returns a different shape. Errors are sometimes `{error: "..."}`,
sometimes `{message}`, sometimes a bare string, sometimes an HTML page.

**Why it breaks:** The frontend needs a branch per endpoint, so error handling gets skipped and
users see nothing when something fails.

**Fix:** One error envelope across the whole API, with a machine-readable code the frontend can
branch on and a human-readable message it can display.

```json
{
  "error": {
    "code": "insufficient_credits",
    "message": "You need 3 more credits to run this.",
    "details": { "required": 5, "available": 2 }
  }
}
```

Include a request id in both the response and the logs so a user-reported failure can be traced.
Never put a stack trace or SQL text in `message` — see `errors-and-resilience.md`.

---

## CORS

**Signal:** `cors()` with no options, `Access-Control-Allow-Origin: *`, or an origin echoed back
from the request header.

```js
app.use(cors());                                       // allows every origin
res.header('Access-Control-Allow-Origin', req.headers.origin);   // allows every origin, with credentials
```

**Why it breaks:** With `credentials: true`, any website a logged-in user visits can call your API
as them and read the response. Reflecting the origin is exactly equivalent to `*` but bypasses the
browser's refusal to combine `*` with credentials — it is strictly worse.

**Fix:** An explicit allowlist of origins, from configuration.

```js
const ALLOWED = (process.env.CORS_ORIGINS ?? '').split(',').filter(Boolean);
app.use(cors({
  origin: (origin, cb) => cb(null, !origin || ALLOWED.includes(origin)),
  credentials: true,
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],
}));
```

Never include `null` in the allowlist — sandboxed iframes send it. CORS is not access control: it
only governs browsers. A curl request ignores it entirely, so the API still needs authentication.

---

## Pagination

**Signal:** List endpoints with no `limit`, or a `limit` the client sets without a cap.

**Why it breaks:** Covered in `data-layer.md` as a database problem; it is also a contract problem.
Clients built against an unpaginated endpoint break when you add pagination later.

**Fix:** Paginate from the start. Return the page plus metadata (`nextCursor`, or `total` and `page`
if the count is cheap). Cap `limit` server-side — accept 100, silently clamp 10000.

---

## Versioning and breaking changes

**Signal:** No version in the path or header. Fields renamed or removed in place.

**Why it breaks:** Mobile clients and third-party integrations do not update on your schedule. A
removed field is an outage for them.

**Fix:** Version from the first public release (`/v1/`). Additive changes are safe; removals and
renames are not. To remove a field: add the replacement, mark the old one deprecated in docs, log
usage, and remove only when usage hits zero. Never change a field's type or meaning in place.

---

## HTTP method semantics

**Signal:** `GET /api/deleteUser?id=5`. State-changing operations behind GET. `POST` used for
everything.

**Why it breaks:** GET is assumed safe by browsers, prefetchers, crawlers, proxies, and link
previewers. A GET that mutates gets triggered by things that never intended to. It is also cached,
logged with its parameters, and CSRF-able from an `<img>` tag.

**Fix:** GET reads only, never mutates. POST creates. PUT/PATCH update. DELETE deletes. PUT and
DELETE should be idempotent — calling twice has the same effect as once.

---

## Security headers

**Signal:** No `helmet`, `django-csp`, `secure_headers`, or equivalent. Responses missing
`X-Content-Type-Options`, `Strict-Transport-Security`, `X-Frame-Options`/`frame-ancestors`.

**Fix:** Set at minimum:

- `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- `X-Content-Type-Options: nosniff`
- `Content-Security-Policy` with `frame-ancestors 'none'` (or `X-Frame-Options: DENY`)
- `Referrer-Policy: strict-origin-when-cross-origin`
- Remove `X-Powered-By` / `Server` version banners

Also enforce HTTPS — redirect HTTP, and behind a proxy set the framework's trusted-proxy setting so
it reads `X-Forwarded-Proto` correctly. Getting that wrong makes the app think every request is
insecure (redirect loop) or secure (no redirect at all).

---

## Content type and body limits

**Signal:** No body size limit configured. `Content-Type` not checked. JSON parser applied to every
route including file uploads and webhooks.

**Why it breaks:** A 500 MB JSON body exhausts memory. A webhook whose raw body was consumed by the
JSON parser can no longer have its signature verified.

**Fix:** Set an explicit body limit (`express.json({ limit: '100kb' })`, `client_max_body_size`,
`DATA_UPLOAD_MAX_MEMORY_SIZE`). Raise it only on the routes that need it. Reject unexpected content
types. Capture the raw body on webhook routes before parsing.

---

## Over-fetching in responses

**Signal:** `res.json(user)`, `return jsonify(record.__dict__)`, `fields = '__all__'`,
`toJSON()` on a model with no allowlist.

**Why it breaks:** Ships `passwordHash`, `resetToken`, `stripeCustomerId`, internal flags, and other
users' emails from included relations. Adding a column later silently starts exposing it.

**Fix:** An explicit output shape per endpoint — a DTO, serializer, or a hand-written object. Deny by
default: new columns are private until someone adds them to the response.

---

## Long-running work

**Signal:** A request handler doing image processing, PDF generation, bulk email, or a large export
inline.

**Why it breaks:** Platform gateways time out (often 30–300s depending on host), and the client sees
a 504 while the work continues invisibly. It also occupies a worker that cannot serve other requests.

**Fix:** Enqueue the work, return `202 Accepted` with a job id, and expose a status endpoint or push
the result over a webhook/websocket. If the platform supports durable workflows or background
functions, use those rather than fire-and-forget promises — an un-awaited promise on serverless is
killed when the response is sent.

---

## Documentation

**Signal:** No OpenAPI spec, no README describing the endpoints, no example requests.

**Fix:** Generate a spec from the code where the framework supports it (FastAPI, NestJS Swagger,
drf-spectacular, Spring springdoc, ASP.NET minimal APIs). It doubles as a contract test: if the spec
cannot describe the endpoint, the endpoint is probably inconsistent. Ensure the docs UI is disabled
or authenticated in production if the API is not public.
