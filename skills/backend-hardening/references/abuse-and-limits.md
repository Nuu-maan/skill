# Abuse, limits, and cost control

Load when the codebase has public endpoints, file uploads, email/SMS sending, or calls a paid
third-party API.

For a solo developer, the realistic threat is usually not a targeted attacker — it is an unbounded
bill or a service that falls over. Both are prevented by the same controls.

## Contents

- No rate limiting
- Rate limiting done wrong
- Enumeration
- Unbounded cost on paid APIs
- File uploads
- Expensive queries and endpoints
- Email and SMS abuse
- Amplification and recursion
- Bot traffic

---

## No rate limiting

**Signal:** No rate-limit middleware anywhere, or one applied globally but not on the routes that
matter.

**Why it breaks:** Credential stuffing against login, unlimited signup spam, password-reset email
floods, scraping of your entire dataset, and one client accidentally DoSing you with a retry loop.

**Fix:** Rate limit these first, harder than everything else:

| Route | Reason |
| :--- | :--- |
| Login, MFA verify | Credential stuffing, code brute force |
| Signup | Fake accounts, resource exhaustion |
| Password reset, magic link, OTP send | Email/SMS cost and inbox spam |
| Anything calling a paid API (LLM, geocoding, SMS) | Direct money |
| File upload | Storage cost, processing cost |
| Search and list endpoints | Scraping, expensive queries |
| Webhook receivers | Forged-event floods |

Then apply a looser global default to everything else. Return `429` with `Retry-After`.

**Choose the key deliberately.** Per-IP alone is wrong for login: an attacker rotates IPs, and users
behind one NAT share a limit. Limit per account **and** per IP, and take the stricter result.

---

## Rate limiting done wrong

**Signal:** An in-memory rate limiter (`express-rate-limit` with the default store, a module-level
`Map`) on a service running more than one instance, or on serverless.

**Why it breaks:** Each instance keeps its own counter, so the effective limit is multiplied by the
instance count. On serverless, a fresh instance starts at zero — the limit barely exists.

**Fix:** A shared store — Redis, Upstash, the platform's rate-limit primitive, or the CDN/WAF layer.
Where the host offers rate limiting at the edge, use it: blocking before the request reaches your
function is cheaper and works during an attack when your app is already saturated.

**Signal:** The limiter keys on `X-Forwarded-For` without a trusted-proxy configuration.

**Why it breaks:** That header is client-supplied. An attacker sets a new value per request and the
limit never triggers. Configure the framework's trusted-proxy setting so it takes the correct hop.

---

## Enumeration

**Signal:** Endpoints that reveal whether a resource exists: signup saying "email already
registered", login distinguishing unknown-user from wrong-password, `GET /users/:id` returning 403
for existing and 404 for missing, sequential integer ids.

**Why it breaks:** An attacker builds a list of your real users, or measures your growth from id
gaps. Combined with any IDOR, sequential ids turn one bug into a full data export.

**Fix:** Identical responses and identical timing for exists/does-not-exist on unauthenticated
endpoints. Handle "email already registered" by sending an email to that address rather than telling
the browser. Opaque ids (UUID/ULID) for anything in a URL. Rate limit the endpoints that leak the
signal.

---

## Unbounded cost on paid APIs

**Signal:** An endpoint that calls an LLM, image generation, geocoding, SMS, or any per-call-priced
API, with no per-user quota, no input size cap, and no spend ceiling.

**Why it breaks:** This is the fastest way for a hobby project to produce a five-figure bill. It does
not require malice — a loop in someone's script is enough. If the key is also exposed to the browser
(`secrets-and-config.md`), it takes hours.

**Fix:**
- Require authentication. Anonymous access to a paid API is an open wallet.
- Per-user quota, enforced in the database (daily and monthly), checked atomically before the call.
- Cap input size — token count, prompt length, image dimensions, batch size.
- Cap output — `max_tokens`, result count.
- Set a hard budget alert **and** a hard spend limit at the provider where available. Alerts tell you
  after the fact; limits stop it.
- Log cost per call per user so abuse is visible.
- Cache identical requests.

---

## File uploads

**Signal:** An upload endpoint with no size limit, no type check, no filename sanitization, files
written into a web-served directory, or a `Content-Type` header trusted as the file type.

**Why it breaks:**
- No size limit fills the disk or the storage bill.
- Trusting the client's `Content-Type` or the extension means an uploaded `.php`/`.jsp` in a served
  directory becomes remote code execution.
- Using the client's filename allows path traversal (`../../app/index.js`) and overwriting.
- Image processing libraries have a long history of parser vulnerabilities; feeding them arbitrary
  bytes is risky.
- A zip/archive extractor without limits is vulnerable to zip bombs and path traversal via entry
  names ("zip slip").

**Fix:**
- Enforce a size limit at the proxy/gateway and in the app.
- Generate the stored filename yourself (UUID); keep the original name as a display-only field.
- Determine the type by inspecting content (magic bytes), not the header or extension, and allowlist
  the result.
- Store outside the web root — ideally in object storage, served through signed URLs, from a
  different domain than your app so a stored HTML/SVG cannot run in your origin.
- Serve with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff` where it does
  not need to render.
- Prefer direct-to-storage presigned uploads so large files never pass through your server — but
  constrain the presigned policy (size, content type, key prefix) and require auth to obtain one.
- For archives: cap entry count, total uncompressed size, and reject entries with absolute or `..`
  paths.

---

## Expensive queries and endpoints

**Signal:** Client-controlled `limit`, `depth`, `include`, `expand`, or filter complexity. A GraphQL
endpoint with no depth or complexity limit. A search endpoint doing a `LIKE '%term%'` scan. A report
endpoint aggregating an entire table.

**Why it breaks:** One request costs the server a thousand times what it costs the caller. That
asymmetry is a denial of service that needs no botnet.

**Fix:** Cap `limit` server-side. Allowlist `include`/`expand` and cap nesting depth. For GraphQL,
enforce depth and complexity limits and disable introspection in production. Put a statement timeout
on the database. Move heavy reports to a background job with a cached result. Rate limit search
separately and more strictly.

---

## Email and SMS abuse

**Signal:** Any endpoint that sends a message to a user-supplied address or number without a limit.
Contact forms, invites, "share this", password reset, OTP.

**Why it breaks:** SMS costs real money per message and is a known fraud target (traffic pumping to
premium numbers). Email floods get your sending domain blacklisted, after which your legitimate
transactional email stops arriving — a slow outage that is hard to reverse.

**Fix:** Rate limit per recipient, per sender, and per IP. Cap sends per account per day. Validate
and normalize phone numbers, and allowlist country codes you actually serve. Require the sender to be
authenticated for anything that names a recipient. Do not let the caller control the message body or
the reply-to address.

---

## Amplification and recursion

**Signal:** A webhook that can be pointed at your own API. A job that enqueues itself. A "sync"
endpoint that triggers another service which calls back. A retry with no attempt counter.

**Why it breaks:** Infinite loops that consume the queue, the database, and the budget, usually
discovered by the bill.

**Fix:** Attempt counters on jobs with a dead-letter queue after N failures. A max depth or a
loop-detection header on anything that can call back into itself. Never let a user-supplied URL point
at your own infrastructure (see SSRF in `input-validation.md`).

---

## Bot traffic

**Signal:** Public signup, comment, or contact endpoints with no bot control. Scraping of a public
API with no limits.

**Fix:** A challenge (CAPTCHA/Turnstile) on signup and unauthenticated write endpoints, or the
platform's bot detection where available. Require email verification before an account can consume
resources. Where the host provides a WAF or bot management, configure it — filtering at the edge is
far cheaper than filtering in your application, and it keeps working when your application is already
overloaded.
