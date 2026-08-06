# Authentication

Who the caller is. Load when the codebase has login, signup, sessions, JWTs, password reset, OAuth,
or API keys.

## Contents

- Password storage
- Login flow
- Sessions and cookies
- JWTs
- Password reset and email verification
- OAuth and social login
- API keys and service-to-service auth
- Multi-factor and account recovery
- Logout and revocation

---

## Password storage

**Signal:** `md5(`, `sha1(`, `sha256(password`, `hashlib.sha`, `crypto.createHash`, storing
`password` on the user model without a hashing call, or any hand-rolled salt/hash loop.

**Why it breaks:** SHA and MD5 are built to be fast, which is exactly wrong for passwords — a
consumer GPU tries billions per second. When the database leaks, every password is recovered and
reused against the users' other accounts.

**Fix:** bcrypt, argon2id, or scrypt. Nothing else. Use the library's default cost, do not lower it
to "make login faster." bcrypt silently truncates at 72 bytes, so pre-hash with SHA-256 first only
if you must support long passphrases.

```js
// Node
const hash = await bcrypt.hash(password, 12);
const ok   = await bcrypt.compare(password, user.passwordHash);
```

```python
# Python
from argon2 import PasswordHasher
ph = PasswordHasher()
hash = ph.hash(password)
ph.verify(user.password_hash, password)   # raises on mismatch
```

**Also check:**
- The password column is not selected by default and never appears in an API response. Grep for
  `SELECT *`, serializers with no field allowlist, and `res.json(user)`.
- No minimum-length check under 8, and no maximum under 64.
- Passwords are not logged. Grep for `console.log(req.body`, `logger.info(request.data`,
  `print(form)` in the auth path.

---

## Login flow

**Signal:** Login returns different responses for "no such user" and "wrong password". No delay, no
lockout, no rate limit on the login route.

**Why it breaks:** Distinct responses let an attacker enumerate which emails have accounts — useful
for targeted phishing and credential stuffing. No rate limit means an attacker can try a common
password against every enumerated email until one works.

**Fix:**
- One generic message for both cases: "Invalid email or password."
- Compare with a constant-time function and always run the hash comparison, even when the user does
  not exist, against a dummy hash. Otherwise response timing leaks existence.
- Rate limit per IP **and** per account. See `abuse-and-limits.md`.
- Lock or step up to a challenge after repeated failures on one account.

**Signal:** Session or token issued before verifying the password, or a `return` missing after a
failure branch so execution falls through to the success path.

**Why it breaks:** Authentication bypass. Read failure branches carefully — a missing `return` or
`else` in a hand-written handler is a real and common Critical.

---

## Sessions and cookies

**Signal:** `httpOnly` absent or false, `secure` absent, `sameSite` absent, session secret hardcoded
or defaulted to `"secret"` / `"keyboard cat"` / `"changeme"`.

**Why it breaks:** Without `httpOnly`, any XSS on the frontend steals the session. Without `secure`,
the cookie travels over plain HTTP. Without `sameSite`, other sites can make authenticated requests
as the user (CSRF). A guessable session secret lets an attacker forge a session for any user.

**Fix:**

```js
cookie: {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'lax',        // 'strict' if no cross-site flows; 'none' requires secure: true
  maxAge: 1000 * 60 * 60 * 24 * 7,
  path: '/',
}
```

**Also check:**
- Session store is not the default in-memory store in production. In-memory loses every session on
  restart and does not work with more than one instance.
- Session ID is regenerated on login (`req.session.regenerate`, `request.session.cycle_key()`,
  `reset_session`). Without this, an attacker who plants a session ID before login owns the session
  after login (session fixation).
- Sessions expire. An unbounded session is a permanent credential.

---

## JWTs

**Signal:** `jwt.decode(` used where `jwt.verify(` was meant. `algorithms` not pinned.
`verify(token, secret, { algorithms: ['none'] })`. Secret shorter than 32 bytes or reused from
another purpose.

**Why it breaks:** `decode` parses without checking the signature — anyone can hand you a token
claiming to be an admin. Not pinning the algorithm allows algorithm-confusion attacks where a token
signed with the public key as an HMAC secret is accepted.

**Fix:**

```js
jwt.verify(token, process.env.JWT_SECRET, {
  algorithms: ['HS256'],   // pin explicitly, always
  issuer: 'your-app',
  audience: 'your-api',
});
```

**Also check:**
- Expiry is set and short (15–60 min for access tokens). `exp` missing means the token never dies.
- The token is not stored in `localStorage` if you can avoid it — an httpOnly cookie is safer against
  XSS. If it must be in JS-readable storage, accept that XSS equals full account takeover and
  prioritize XSS defenses accordingly.
- No sensitive data in the payload. A JWT is signed, not encrypted; anyone can read it.
- **There is a revocation story.** Stateless JWTs cannot be invalidated. If a user logs out, changes
  their password, or is banned, their token still works until expiry. Either keep expiry very short
  with refresh tokens, or check a server-side denylist/version counter per user.
- Refresh tokens are stored server-side, rotated on use, and reuse of an old refresh token
  invalidates the whole family (that is how you detect theft).

---

## Password reset and email verification

**Signal:** Reset token generated with `Math.random()`, `random.random()`, `rand()`, a timestamp, or
a sequential id. Token with no expiry. Token reusable after use. Reset endpoint that accepts a
user id or email in the body and trusts it.

**Why it breaks:** A predictable token means an attacker resets any account. An endpoint that takes
`{ email, newPassword }` and does not require a token is a direct account-takeover primitive — and
this exact shape appears often in AI-generated code.

**Fix:**
- Generate with a CSPRNG: `crypto.randomBytes(32)`, `secrets.token_urlsafe(32)`,
  `SecureRandom.urlsafe_base64(32)`, `crypto/rand`.
- Store a **hash** of the token, compare hashes. A leaked database should not hand over live tokens.
- Expire in 15–60 minutes. Single use — delete on consumption.
- Invalidate all existing sessions when the password changes.
- Return the same response whether or not the email exists.
- Rate limit the request-reset endpoint; it sends email and is a spam/cost vector.

**Signal:** Reset link built from a request header — `req.headers.host`, `X-Forwarded-Host`.

**Why it breaks:** Host header poisoning. An attacker requests a reset for the victim with a forged
`Host`, and the victim's email contains a link to the attacker's domain carrying the real token.

**Fix:** Build links from a configured `APP_URL` constant, never from the request.

---

## OAuth and social login

**Signal:** No `state` parameter, or `state` generated but never validated on the callback. Provider
`redirect_uri` allowlist containing wildcards or `localhost` in production. Trusting an email from
the provider without checking `email_verified`.

**Why it breaks:** Missing `state` is CSRF on the login flow — an attacker can link their provider
account to the victim's session. Trusting unverified emails lets an attacker register a provider
account with the victim's email and take over the matched local account.

**Fix:** Generate `state` per attempt, store it in the session, compare and consume on callback.
Use PKCE for public clients. Match accounts on the provider's stable subject id, not on email.
Only auto-link by email when the provider asserts the email is verified.

---

## API keys and service-to-service auth

**Signal:** API key compared with `==` / `===`. Key stored in plaintext. One shared key for all
customers. Key accepted from a query string.

**Why it breaks:** Non-constant-time comparison can leak the key byte by byte through timing. Query
strings land in access logs, proxies, and browser history.

**Fix:** Store a hash of the key, compare with a constant-time function (`crypto.timingSafeEqual`,
`hmac.compare_digest`, `subtle.ConstantTimeCompare`). Accept keys in the `Authorization` header. Give
each consumer its own key with its own scopes so one can be revoked without breaking the rest. Prefix
keys with an identifiable string (`myapp_live_...`) so secret scanners can catch leaks.

**Signal:** A webhook endpoint that does not verify a signature.

**Why it breaks:** Anyone who learns the URL can post fake events. For a payments webhook this means
free products; for a provisioning webhook it means free accounts.

**Fix:** Verify the provider's signature over the **raw** body before parsing. Many frameworks parse
JSON before your handler runs and the re-serialized body will not match the signature — capture the
raw body for that route specifically. Reject stale timestamps to prevent replay.

---

## Multi-factor and account recovery

**Signal:** TOTP code verified without checking it has not already been used. No limit on TOTP
attempts. Recovery codes stored in plaintext. MFA enforced on login but not on password change,
email change, or MFA disable.

**Why it breaks:** An unlimited-attempt 6-digit code falls in minutes. Recovery paths that skip MFA
make the MFA decorative.

**Fix:** Rate limit code attempts hard (5 per window). Record the last accepted time step and reject
replays. Hash recovery codes and consume them on use. Re-authenticate before any security-sensitive
change: password, email, MFA settings, API keys.

---

## Logout and revocation

**Signal:** Logout clears the cookie client-side only, or deletes it without destroying the
server-side session. No way to sign out other devices.

**Why it breaks:** A stolen session survives the user's attempt to kill it.

**Fix:** Destroy the server session record, clear the cookie with the same attributes it was set
with, and increment a per-user token version so outstanding JWTs stop validating. Offer a
"sign out everywhere" that bumps that version.
