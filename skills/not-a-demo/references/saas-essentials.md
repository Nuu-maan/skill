# SaaS essentials

What real apps have that demos skip. Tags: **Always** / **When** / **Scale**. Use the framework's or a provider's auth rather than hand-rolling it, then make sure every item below is covered.

## Accounts and auth (Always, if there are accounts)

- The full flow, on the server: sign up, log in, log out (kills the server session), password reset, email verification. Not just a login form.
- Password rules and hashing: see [security.md](security.md). Add a show-password toggle (it replaces the confirm-password field) and `autocomplete="new-password"` / `"current-password"`.
- Password reset: the same response whether or not the account exists. A random, single-use token that expires (15-60 min). Build the link from your configured base URL, never the Host header. `Referrer-Policy: no-referrer` on the reset page. After a reset, end other sessions and don't log the user in automatically.
- Ask for the current password before changing email, password or payment method. Notify the old address when the email changes.
- Corporate email scanners open links before users do and burn single-use tokens. Magic, verification and invite links land on a page with a confirm button, or use a 6-digit code instead.
- OAuth (When): authorization code flow with PKCE, a `state` parameter, exact redirect URIs. Link an OAuth sign-in to an existing account by email only if that email is verified; otherwise attackers pre-register a victim's email and take over the account.
- An active-sessions list with "Log out other sessions". Email on a new-device login (Scale).

## Settings and data (Always, if there are accounts)

- An account page: name, email, password, active sessions, delete account.
- Account deletion is self-serve, really deletes or anonymizes the data, and says how long it takes. Apple requires in-app deletion for iOS apps.
- Data export (When serving EU/UK users): the user's data as CSV or JSON.
- Teams (When): invite by email, roles, remove a member, transfer ownership, leave the team.

## Billing (When there are payments)

- Use a hosted flow (Stripe Checkout, Billing portal or similar). Never store card data.
- Webhooks are the source of truth: verify the signature, dedupe by event ID, don't assume event order, return 2xx fast, process in a job.
- Handle `invoice.paid` (extend access only if the subscription is active), `invoice.payment_failed` (email the customer, enable smart retries), `customer.subscription.trial_will_end` (arrives 3 days before), `past_due` (warn), `canceled`/`unpaid` (revoke), refunds and disputes. Keep an access-until timestamp in your own database.
- Money is an integer in minor units plus a currency code (`1000` = $10.00; JPY has no minor unit). Never floats.
- Idempotency keys on POSTs that charge money or have side effects: the client sends a key, the server stores key + params + response, returns 409 on a conflicting reuse, and expires keys after about 24h.
- A billing page: current plan, usage, next invoice date, payment method, past invoices, and cancel without dark patterns.

## Email (Always, if you send email)

- Send from a background job through a provider (Postmark, Resend, SES, SendGrid).
- The sending domain has SPF, DKIM and DMARC. Bulk senders need one-click unsubscribe on marketing mail and a spam rate under 0.3%.
- Transactional emails are plain and specific: a subject that says what happened ("Your invoice from Acme"), one action, a plain-text version.
- A welcome email with the one next step. Transactional and marketing mail kept separate; users can unsubscribe from marketing.

## Dates, time, text (Always)

- Store instants in UTC and show them in the user's time zone. Future local events (a 9am meeting) store local time + IANA zone ID.
- Relative time for recent things ("5 min ago"), absolute with the year for old ones, the full timestamp on hover.
- UTF-8 everywhere, `lang` on `<html>`, `Intl` for numbers, dates and currency. No string concatenation for sentences with variables; it breaks plurals and translation.

## Data safety (Always)

- Automated backups, a restore you have actually tested, and an alert when a backup fails. Replication isn't a backup. GitLab lost 6 hours of data in 2017 because backups had been failing silently.
- Migrations that don't lock big tables: add columns as nullable, backfill in batches, create indexes concurrently, rename by expand-and-contract, set a lock timeout. Tools: strong_migrations (Rails), squawk (Postgres).
- Soft-delete content people might want back. Hard-delete on account deletion.
- Audit log (When B2B or teams): who did what to what, when, from where. Append-only and exportable.

## Operations

- One structured log line per request: request ID, user ID, route, status, duration, DB time (Always).
- Error tracking (Sentry or similar) with source maps, wired up before launch (Always).
- Health checks: liveness checks only the process; readiness may check dependencies (Scale).
- Feature flags (When): few, short-lived, deleted after release. Keep kill switches.
- Boring tech by default: Postgres or SQLite, the framework's built-in jobs and cache before adding Redis.
- Public API (Scale): version it. Adding fields is safe; renaming fields or changing types breaks clients.

## Launch basics (Always, for anything public)

- Favicons: `favicon.ico` (32px), `icon.svg`, `apple-touch-icon.png` (180px), manifest icons at 192 and 512px.
- Every page has a unique `<title>` and meta description. Open Graph: `og:title`, `og:description`, `og:image` (1200×630), `og:url`, plus `twitter:card`.
- `sitemap.xml`, `robots.txt`, canonical URLs on public pages.
- Custom 404 and 500 pages.
- Privacy policy and terms linked in the footer (required for Google OAuth verification and the app stores). Cookie consent if you set non-essential cookies for EU users.
- A real support contact (email or help link) reachable from inside the app.
