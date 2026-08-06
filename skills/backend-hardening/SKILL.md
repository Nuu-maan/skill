---
name: backend-hardening
description: Audits a backend or API for security vulnerabilities, stability bugs, and design flaws, explains each one in plain language, then fixes them on request. Covers authentication, authorization and IDOR, injection, input validation, exposed secrets, database and transaction correctness, race conditions, error handling, rate limiting, and deployment config across any language or framework including Node/Express/Nest/Fastify/Next, Python/Django/FastAPI/Flask, PHP/Laravel, Ruby/Rails, Go, Java/Spring, .NET, and Supabase/Firebase. Use whenever the user asks to review, audit, secure, harden, or production-proof a backend or API; asks whether their backend is safe, secure, or ready to launch; mentions vulnerabilities, auth bugs, SQL injection, exposed API keys, or leaked secrets; is about to deploy a backend; or has AI-generated backend code they do not fully understand. Also use proactively when writing or editing server-side route handlers, database queries, or auth logic.
license: MIT
compatibility: Requires filesystem read access and a code-search tool such as grep or ripgrep. Optional shell access to run the project's package manager, tests, linter, and dependency audit improves accuracy.
allowed-tools: Read, Grep, Glob
metadata:
  version: 1.0.0
  author: nuu-maan
  homepage: https://qala.lol
---

# Backend Hardening

Audit a backend, report what is actually broken, then fix it with the user's approval.

**Assume the user is strong at frontend and weak at backend.** They likely generated much of this
code with an AI and cannot judge whether a finding is real. That has three consequences:

1. **A false positive is worse than a miss.** They cannot filter your output. Never report a finding
   you have not confirmed by reading the actual code path.
2. **Every finding needs a plain-language "so what."** State the concrete bad outcome — "any logged-in
   user can read every other user's invoices by changing the id in the URL" — not the category name.
3. **Never edit code without asking.** Report first. Fix second, on their word.

## Operating rules

- Every finding cites `file:line` and quotes the offending code. No finding without evidence.
- Verify before reporting: trace the request path and confirm nothing upstream (middleware, a guard,
  a framework default) already prevents it. Drop anything you cannot confirm.
- Never invent framework behavior. If unsure whether a framework escapes something by default, say so
  and mark the finding as needing confirmation rather than asserting it.
- Report in severity order. A beginner will fix the top item and stop, so the top item must matter most.
- Do not rewrite architecture. Fix the defect in front of you.
- Never print a real secret you find. Show `STRIPE_SECRET_KEY=sk_live_4eC***` and the location only.

## Workflow

Copy this checklist into your reply and tick items as you go:

```
Backend audit:
- [ ] Phase 1: Map the stack
- [ ] Phase 2: Fast sweep (highest-signal checks)
- [ ] Phase 3: Deep audit (load only relevant reference files)
- [ ] Phase 4: Verify every candidate finding
- [ ] Phase 5: Report, ranked by severity
- [ ] Phase 6: Offer to fix, then fix in priority order
```

### Phase 1: Map the stack

Do not audit blind. Identify these first, from files rather than from guessing:

| What | Where to look |
| :--- | :--- |
| Language + framework | `package.json`, `requirements.txt`, `pyproject.toml`, `go.mod`, `composer.json`, `Gemfile`, `pom.xml`, `build.gradle`, `*.csproj` |
| Entrypoint + routes | `main.*`, `app.*`, `server.*`, `index.*`, `routes/`, `controllers/`, `api/`, `handlers/` |
| Database + access layer | ORM config, migration folder, `schema.prisma`, `models/`, raw SQL strings |
| Auth mechanism | Session middleware, JWT libs, `passport`, `next-auth`, `django.contrib.auth`, Devise, Spring Security, Supabase/Firebase client |
| Hosting + runtime | `Dockerfile`, `vercel.json`, `vercel.ts`, `fly.toml`, `render.yaml`, `serverless.yml`, CI workflows |
| Trust boundary | Which routes are public, which require auth, which require a role |

Then load the matching stack file — **one only**, whichever fits:

- Node / TypeScript (Express, Nest, Fastify, Hono, Next.js route handlers) → [stacks/node.md](stacks/node.md)
- Python (Django, FastAPI, Flask) → [stacks/python.md](stacks/python.md)
- PHP (Laravel, Symfony, plain PHP) → [stacks/php.md](stacks/php.md)
- Ruby (Rails, Sinatra) → [stacks/ruby.md](stacks/ruby.md)
- Go (net/http, Gin, Echo, Fiber, Chi) → [stacks/go.md](stacks/go.md)
- Java/Kotlin (Spring Boot) or C# (ASP.NET Core) → [stacks/jvm-dotnet.md](stacks/jvm-dotnet.md)
- Supabase, Firebase, Appwrite, PocketBase, or any "no backend" setup → [stacks/baas.md](stacks/baas.md)

If the project uses a BaaS **and** a custom server, load `stacks/baas.md` plus the matching language file.

### Phase 2: Fast sweep

These twelve checks catch the majority of real defects in AI-assisted backends. Run them before
anything else, on every audit, regardless of stack.

1. **Secrets in the repo.** Grep for `sk_live`, `sk_test`, `AKIA`, `-----BEGIN`, `password =`,
   `api_key`, `token =`, `SECRET`. Check `.env`, `config/*`, and whether `.env` is in `.gitignore`.
   Check git history if available, not just the working tree.
2. **Secrets shipped to the browser.** Any server key behind a `NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`,
   `EXPO_PUBLIC_`, or `PUBLIC_` prefix is public. So is anything imported into a client component.
3. **Routes with no auth check.** List every route. For each, find the line that enforces
   authentication. Any route that mutates data or returns user data without one is a finding.
4. **Ownership checks (IDOR).** For every handler that reads an id from the URL, body, or query:
   does the query filter by the *current user* as well as the id? `findById(params.id)` with no
   owner clause is the single most common serious bug in this class of code.
5. **Raw SQL built by string concatenation or interpolation.** Grep for `SELECT`/`INSERT`/`UPDATE`/
   `DELETE` adjacent to `+`, `f"`, `${`, `%s` used as formatting, `.format(`, `#{`, or `"` + var.
6. **Unvalidated request bodies.** Does anything validate shape and type before use? A handler that
   reads `req.body.role` or spreads the whole body into a database write is a finding.
7. **Mass assignment.** `Object.assign(user, req.body)`, `{...req.body}`, `Model(**request.data)`,
   `update_attributes(params[:user])`, `.map(dto)` with no allowlist — all let a caller set fields
   they should not, like `role`, `isAdmin`, `credits`, `verified`.
8. **Errors leaking internals.** Stack traces, SQL text, or raw exception messages returned to the
   client. Debug mode on in production.
9. **No rate limit on auth or expensive routes.** Login, signup, password reset, OTP, search,
   file upload, and anything calling a paid API.
10. **Passwords.** Stored with bcrypt/argon2/scrypt, or with MD5/SHA/plaintext? Any custom crypto?
11. **CORS.** `Access-Control-Allow-Origin: *` combined with credentials, or an origin reflected
    from the request, or a wildcard on an authenticated API.
12. **Dependencies.** Run the ecosystem audit (`npm audit`, `pip-audit`, `bundle audit`,
    `govulncheck`, `dotnet list package --vulnerable`) if shell access is available.

Anything the sweep hits, confirm in Phase 4 before it becomes a finding.

### Phase 3: Deep audit

Load **only the reference files that match what this codebase actually does.** Each file is a
checklist of signal → why it breaks → fix. Skipping irrelevant files is the point; they cost nothing
until read.

| Load this | When the codebase has |
| :--- | :--- |
| [references/authentication.md](references/authentication.md) | Login, signup, sessions, JWTs, password reset, OAuth, API keys |
| [references/authorization.md](references/authorization.md) | Per-user data, roles, admin routes, teams, multi-tenancy |
| [references/input-validation.md](references/input-validation.md) | Any request body, query param, path param, header, or webhook |
| [references/data-layer.md](references/data-layer.md) | A database, ORM, migrations, transactions, or background jobs |
| [references/api-design.md](references/api-design.md) | HTTP endpoints consumed by a frontend or third party |
| [references/errors-and-resilience.md](references/errors-and-resilience.md) | Calls to other services, queues, retries, long-running work |
| [references/secrets-and-config.md](references/secrets-and-config.md) | Environment variables, config files, multiple environments |
| [references/abuse-and-limits.md](references/abuse-and-limits.md) | Public endpoints, uploads, email/SMS, or paid third-party APIs |
| [references/operations.md](references/operations.md) | A deployment target, logs, health checks, backups |

### Phase 4: Verify

For each candidate finding, before it goes in the report:

1. Read the full handler, not the matched line.
2. Walk backwards through middleware, decorators, guards, filters, and base classes. Many findings
   die here because a global guard already handles it.
3. Confirm the input is genuinely attacker-controlled and reaches the sink.
4. Confirm the framework does not already prevent it by default.
5. If you still cannot prove it, either drop it or label it explicitly as "unconfirmed — needs a
   human to check X." Never present a guess as a vulnerability.

Then rate severity:

| Severity | Definition |
| :--- | :--- |
| **Critical** | Unauthenticated attacker takes over accounts, reads/writes the whole database, executes code, or drains money. Fix before deploying. |
| **High** | Authenticated attacker reaches other users' data or privileges, or the app corrupts/loses data under normal use. |
| **Medium** | Requires unusual conditions, or degrades reliability rather than breaching it — missing timeouts, no rate limit on a cheap route, weak logging. |
| **Low** | Hardening and hygiene. Real, but nothing breaks today. |

Severity is about impact and reachability, not how ugly the code is. A missing index is not High.
A public admin route is not Medium.

### Phase 5: Report

Use the exact structure in [templates/report.md](templates/report.md). Lead with a one-line verdict
and a count by severity so the user knows immediately whether they can ship.

### Phase 6: Offer to fix

End the report with a concrete offer, not an open question:

> I can fix these. Suggested order: the 2 Critical findings first (about 20 lines of changes across
> 3 files), then the 4 High. Want me to start with the Criticals?

Then follow [references/fixing-safely.md](references/fixing-safely.md): smallest correct change,
one concern at a time, verify after each, never silently widen scope.

## When there is nothing to find

Say so plainly and show your work — which routes you enumerated, which checks you ran, what you could
not verify without running the app. "I found no Critical or High issues" is a valid, useful result.
Do not manufacture Medium findings to fill the report.

## Additional resources

- [references/fixing-safely.md](references/fixing-safely.md) — how to apply fixes without breaking things
- [templates/report.md](templates/report.md) — the report format to follow
