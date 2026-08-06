# Operations

Running it, seeing it, and keeping it running. Load when the codebase has a deployment target, logs,
health checks, or third-party dependencies.

## Contents

- Logging
- Logging secrets and PII
- Metrics and error tracking
- Tracing a user report
- Dependencies and supply chain
- Container and runtime hardening
- Deployment safety
- Database access from outside
- Documentation and the bus factor

---

## Logging

**Signal:** `console.log` / `print` as the only logging. No structure, no levels, no timestamps, no
request id.

**Why it breaks:** When something breaks in production you have a wall of unsearchable text with no
way to correlate lines belonging to one request. Debugging becomes guesswork.

**Fix:** A structured logger (pino, winston, structlog, zap, Serilog, Monolog) emitting JSON with a
level, timestamp, and a request id attached to every line in a request's lifetime. Generate the
request id at the edge, accept an inbound one if present, propagate it to downstream calls, and
return it to the client so a user can quote it.

Log at boundaries: request received (method, path, status, duration), outbound call (target, status,
duration), and every error with its stack. Do not log inside tight loops. Do not log successful
reads of every row.

**Signal:** No log retention, or logs only on the instance's local disk.

**Why it breaks:** The container is replaced and the evidence of the incident goes with it.

**Fix:** Ship logs off the instance. Set a retention period that matches how long you might take to
notice a problem.

---

## Logging secrets and PII

**Signal:** `logger.info(req.body)`, `print(request.headers)`, `console.log(user)`, logging a full
error object that contains the SQL parameters or the request that caused it.

**Why it breaks:** Passwords, tokens, `Authorization` headers, card details, and personal data land
in a log aggregator that has broader access than the database and often longer retention. This is
frequently the actual location of a breach.

**Fix:** An explicit redaction list in the logger config — `password`, `token`, `authorization`,
`cookie`, `secret`, `apiKey`, `ssn`, `card`. Log identifiers, not payloads: log `userId`, not the
user object. Assume anything logged is readable by everyone with dashboard access.

---

## Metrics and error tracking

**Signal:** No error tracker. No alerting. The way you find out about outages is a user message.

**Fix:** An error tracker (Sentry or equivalent) capturing unhandled exceptions with the request id
and release version, so a regression is traceable to a deploy. At minimum, alert on: error rate,
p95 latency, 5xx count, and — for anything with a bill — spend. Every alert must be actionable; an
alert nobody acts on trains everyone to ignore alerts.

Health of the dependencies matters as much as the app: database connection count against the max,
queue depth, and job failure rate.

---

## Tracing a user report

**Signal:** No way to answer "user X says checkout failed at 3pm" without reading code.

**Fix:** Make sure these three are linked: the request id in the client's error message, the log
lines for that request, and the error tracker event. If a support message with a request id cannot be
resolved to a single log trace in under a minute, the observability is not finished.

---

## Dependencies and supply chain

**Signal:** No lockfile committed. Versions pinned with `*` or `latest`. No audit ever run. Direct
dependencies far behind. Packages with names close to popular ones (typosquats). Install scripts from
unfamiliar packages.

**Why it breaks:** Most of the code you ship is not yours. A known-vulnerable transitive dependency
is exploitable in your app; a compromised package with a postinstall script exfiltrates your
environment at build time.

**Fix:**
- Commit the lockfile. Install with the reproducible command in CI (`npm ci`, `pip install -r` with
  hashes, `bundle install --deployment`, `go mod verify`).
- Run the audit in CI and fail on high severity:
  `npm audit --audit-level=high`, `pip-audit`, `bundle audit`, `govulncheck ./...`,
  `dotnet list package --vulnerable --include-transitive`, `mvn dependency-check`.
- Enable automated dependency updates so patches are routine rather than an event.
- Review what a new dependency actually is before adding it — download counts, repository, last
  release, and whether it is needed at all.

---

## Container and runtime hardening

**Signal:** `FROM node:latest`, running as root, secrets baked into image layers with `ENV`, no
`.dockerignore` so `.env` and `.git` are copied in, a huge base image.

**Why it breaks:** `latest` makes builds non-reproducible. Root in the container means a code
execution bug becomes container-level control. `ENV SECRET=...` is visible in `docker history` to
anyone who can pull the image, and `.env` copied into the image ships your credentials with it.

**Fix:** Pin the base image to a specific minor version, prefer slim/alpine/distroless. Create and
`USER` a non-root account. Multi-stage build so build tooling and source do not ship. A
`.dockerignore` covering `.env*`, `.git`, `node_modules`, and test fixtures. Inject secrets at
runtime, never at build. Set a read-only root filesystem and drop capabilities where the platform
supports it. Scan the image (`trivy`, `grype`) in CI.

---

## Deployment safety

**Signal:** Deploys straight to production from a laptop. No CI checks. No way to roll back. No
staging environment. Migrations applied automatically at container start by every instance
simultaneously.

**Fix:** Deploy from CI on a green build (tests, lint, typecheck, audit). Keep a one-command rollback
and know that it works — test it once deliberately. Run migrations as a distinct step that happens
once, not from every starting instance. Use the platform's gradual rollout if it has one. Have a
preview/staging environment with its own data.

Confirm the production checklist is genuinely true: debug off, correct log level, HTTPS enforced,
correct CORS origins, real secrets from the secret store, health checks wired to the load balancer,
error tracking receiving events, backups running.

---

## Database access from outside

**Signal:** A database reachable from `0.0.0.0/0`. An admin panel (pgAdmin, phpMyAdmin, Adminer,
Mongo Express, Redis Commander) exposed publicly. Redis or MongoDB with no authentication. Default
ports open to the internet.

**Why it breaks:** Unauthenticated Redis and MongoDB instances are found and wiped for ransom within
hours of being exposed — this is automated and continuous.

**Fix:** Databases on a private network, reachable only by the app. Access for humans via a bastion,
VPN, or the provider's authenticated console. Authentication and TLS enabled. Admin tools not
deployed to production at all; if they must exist, put them behind authentication and an IP
allowlist. Verify by scanning your own public IP for open ports.

---

## Documentation and the bus factor

**Signal:** No README describing how to run the project, no list of required environment variables,
no note on how to deploy or roll back.

**Why it breaks:** For a solo project this is the failure mode that actually bites — six months
later, nobody, including you, can safely change anything.

**Fix:** A README with: prerequisites, how to run locally, the environment variables and what each
does (a maintained `.env.example` with placeholder values), how to run migrations, how to run tests,
how to deploy, how to roll back, and where the logs and error tracker live. Keep it in the repo so it
is updated with the code.
