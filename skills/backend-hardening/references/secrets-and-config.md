# Secrets and configuration

Load when the codebase uses environment variables, config files, or has more than one environment.

## Contents

- Secrets committed to the repository
- Secrets shipped to the browser
- Hardcoded secrets in source
- Weak or default secrets
- Config not validated at boot
- Unsafe defaults
- Environment separation
- Rotation and revocation
- What to do about an already-leaked secret

---

## Secrets committed to the repository

**Signal:** A `.env` file tracked by git. `.env` missing from `.gitignore`. Credentials in
`config/*.json`, `docker-compose.yml`, `terraform.tfvars`, CI workflow files, or a `.pem`/`.p12` in
the tree. A `.env.example` containing real values rather than placeholders.

Grep for: `sk_live`, `sk_test`, `rk_live`, `AKIA`, `ASIA`, `AIza`, `ghp_`, `github_pat_`, `xoxb-`,
`SG.`, `-----BEGIN`, `PRIVATE KEY`, `password=`, `passwd`, `api_key`, `apikey`, `secret`, `token=`,
`Bearer `, `mongodb+srv://`, `postgres://`, `mysql://`, `redis://`.

**Why it breaks:** Public repositories are scraped continuously — leaked cloud keys are typically
abused within minutes, most often for crypto mining billed to the owner. A private repo is not
safe either: every collaborator, every fork, and every future open-sourcing exposes it.

**Fix:** Remove the secret from the working tree, add the file to `.gitignore`, and **rotate the
secret** — deletion from the tree does not remove it from history. See the last section.

Check git history, not just the current files:

```bash
git log --all --full-history -- .env
git log -p --all -S 'sk_live' | head -50
```

Add a secret scanner to CI (`gitleaks`, `trufflehog`, GitHub push protection) so the next one is
blocked before it lands.

---

## Secrets shipped to the browser

**Signal:** A server-side secret behind a public-prefixed variable:

```
NEXT_PUBLIC_STRIPE_SECRET_KEY=...
VITE_SUPABASE_SERVICE_ROLE_KEY=...
REACT_APP_OPENAI_API_KEY=...
EXPO_PUBLIC_DATABASE_URL=...
PUBLIC_ADMIN_TOKEN=...
```

Also: a secret imported into a file that runs on the client, or passed into a component's props.

**Why it breaks:** These prefixes mean "inline this value into the JavaScript bundle." Anyone can
open devtools and read it. A service-role key or an LLM API key exposed this way means unlimited
access and unlimited billing on your account.

**Fix:** Anything secret is read only in server code — a route handler, server action, or backend
service — and never given a public prefix. If the frontend needs the capability, add a backend
endpoint that performs the call with the secret and enforces auth and rate limits.

Only these belong behind a public prefix: publishable/anon keys explicitly documented as public,
public URLs, and feature flags. Check the built bundle to be sure:

```bash
grep -rE 'sk_live|service_role|-----BEGIN' dist/ build/ .next/static/
```

---

## Hardcoded secrets in source

**Signal:** A literal key, password, or connection string in a `.js`/`.py`/`.go`/`.java` file. A
fallback in the config read:

```js
const secret = process.env.JWT_SECRET || 'dev-secret';
```

**Why it breaks:** The fallback is the dangerous part — if the variable is missing in production the
app does not fail, it quietly runs with a secret that is published in your repository. Every token it
signs is forgeable.

**Fix:** Read from the environment with no fallback, and fail to start when it is missing (next
section). Test files and seed scripts count — a test fixture with a real key is a leaked key.

---

## Weak or default secrets

**Signal:** `secret: 'secret'`, `'changeme'`, `'keyboard cat'`, `SECRET_KEY = 'django-insecure-...'`,
a JWT secret shorter than 32 bytes, the same secret reused for sessions, JWTs, and CSRF, or a
framework's generated development key still in place.

**Why it breaks:** Default and short secrets are brute-forceable or simply known. Forging a session
or a JWT is then trivial. Reusing one secret across purposes means one leak breaks everything.

**Fix:** Generate independently, at least 32 bytes from a CSPRNG:

```bash
openssl rand -base64 48
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

One secret per purpose. Different values per environment.

---

## Config not validated at boot

**Signal:** `process.env.X` read at the point of use, deep inside a handler.

**Why it breaks:** A missing variable becomes `undefined` and surfaces as a confusing runtime error
on one code path, in production, hours after deploy — or worse, as a silent misbehavior (an empty
API key sent as a header, a `Bearer undefined`).

**Fix:** Parse and validate the entire environment once, at startup, and crash immediately if
anything is missing or malformed. Import the validated object everywhere else.

```ts
// config.ts — imported by everything, process.env referenced nowhere else
const Env = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET:   z.string().min(32),
  NODE_ENV:     z.enum(['development', 'test', 'production']),
  PORT:         z.coerce.number().default(3000),
}).parse(process.env);

export default Env;
```

Failing at boot is the goal: the deploy fails visibly instead of the app running broken.

---

## Unsafe defaults

**Signal:** Config that defaults to the permissive option — debug on, CORS open, TLS verification
off, auth optional, rate limiting disabled unless enabled.

**Why it breaks:** Defaults are what you get when someone forgets. They should fail closed.

**Fix:** Invert every default. Debug off unless explicitly on. Auth required unless a route is
explicitly public. Never `rejectUnauthorized: false`, `verify=False`, or
`InsecureSkipVerify: true` — that disables certificate checking and makes TLS decorative. If a
self-signed certificate must be trusted, add that specific CA rather than turning verification off.

---

## Environment separation

**Signal:** One database used by development and production. Production credentials in a local
`.env`. Test data written to production. The same third-party account (Stripe, email, storage) across
environments.

**Why it breaks:** A local script with a typo deletes production rows. A test run emails real
customers. There is no way to experiment safely.

**Fix:** Separate credentials, databases, buckets, and third-party accounts per environment. Test
mode keys in non-production. Developers should not hold production credentials by default. Make the
environment visible in logs and in any admin UI so nobody acts on the wrong one.

---

## Rotation and revocation

**Signal:** Keys with no rotation history. No inventory of what secrets exist and who can read them.
Shared credentials with no per-service identity.

**Fix:** Keep secrets in a manager (platform env store, Vault, AWS/GCP Secrets Manager, Doppler) with
access control and an audit trail, not in files passed around. Give each service its own credential
so one can be revoked in isolation. Prefer short-lived, automatically issued credentials (OIDC
federation to a cloud provider) over long-lived static keys where the platform supports it.

---

## What to do about an already-leaked secret

Order matters. State it plainly to the user — this is urgent and rotation comes first.

1. **Rotate the credential at the provider now.** Issue a new one, deploy it, then revoke the old.
   Removing it from git does not un-leak it; anyone who cloned or scraped the repo has it.
2. **Check for abuse.** Provider audit logs, cloud billing, unexpected resources, unfamiliar IPs.
   Leaked cloud keys are exploited fast.
3. **Remove it from the working tree** and add the file to `.gitignore`.
4. **Purge history if the repo is or will be public** (`git filter-repo`, BFG). This rewrites
   history, so coordinate with anyone else working on the repo. Note that forks and any cached
   provider views may retain the old objects — this is cleanup, not containment.
5. **Add scanning** so it cannot recur.

Never skip step 1 in favour of step 4. A rewritten history with a live key is still a live key.
