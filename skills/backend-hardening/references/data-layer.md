# Data layer

Correctness and durability of what you store. Load when the codebase has a database, ORM,
migrations, transactions, or background jobs.

## Contents

- Missing transactions
- Race conditions and lost updates
- Idempotency
- Connection pool exhaustion
- N+1 queries
- Missing indexes
- Unbounded queries
- Schema and constraints
- Migrations
- Soft deletes
- Timezones and money
- Backups and recovery

---

## Missing transactions

**Signal:** Two or more writes that must both succeed, executed as separate statements with no
transaction wrapping them.

```js
await Account.decrement(from, amount);
await Account.increment(to, amount);        // if this throws, money vanished
```

**Why it breaks:** Any failure between the writes leaves the database in a state your code believes
is impossible. Half-created users, orders with no line items, charged cards with no order, inventory
decremented for a purchase that failed.

**Fix:** Wrap the unit of work in a transaction; every write in that unit uses the transaction handle.

```js
await db.transaction(async (tx) => {
  await tx.account.update({ where: { id: from }, data: { balance: { decrement: amount } } });
  await tx.account.update({ where: { id: to   }, data: { balance: { increment: amount } } });
});
```

**Check that the transaction handle is actually used.** A common bug is opening a transaction and
then calling a repository function that uses the global connection — those writes are outside the
transaction and will not roll back.

**Also check:**
- External side effects inside a transaction. Sending an email or charging a card inside a
  transaction that later rolls back means the side effect happened anyway. Do external work after
  commit, or record intent in the transaction and process it afterwards.
- Long transactions holding locks across a network call. Keep them short.
- A transaction opened and never committed or rolled back on the error path — that leaks a
  connection and holds locks until timeout.

---

## Race conditions and lost updates

**Signal:** Read-then-write without protection.

```js
const user = await User.findById(id);
user.credits -= 1;
await user.save();                        // two concurrent requests both spend the same credit
```

Also: "check the count, then insert" for limited stock, seats, invites, or coupon redemptions;
"check if exists, then create" for signup.

**Why it breaks:** Between the read and the write, another request did the same thing. The second
write overwrites the first. Users double-spend credits, oversell stock, redeem one coupon twice, or
end up with duplicate accounts. This is invisible in testing and appears immediately under real
traffic.

**Fix — pick the one that fits:**

1. **Atomic database operation.** Best when possible.
   ```sql
   UPDATE users SET credits = credits - 1 WHERE id = $1 AND credits > 0 RETURNING credits;
   ```
   Zero rows returned means there were no credits. No read needed.

2. **Unique constraint.** For "must not exist twice" — a unique index on `email`, or on
   `(user_id, coupon_id)`. Let the database reject the duplicate and catch the constraint error.
   A pre-check in application code is not a constraint.

3. **Row lock.** `SELECT ... FOR UPDATE` inside a transaction when you need to read, compute, and
   write.

4. **Optimistic locking.** A `version` column; `UPDATE ... WHERE version = $expected`; retry on zero
   rows affected.

Application-level mutexes do not work across processes or instances. If the app runs more than one
copy — and on any serverless platform it does — the guarantee must live in the database.

---

## Idempotency

**Signal:** A `POST` that creates a charge, order, or account with no idempotency key. Webhook
handlers with no dedupe on event id. Retried jobs with no completion marker.

**Why it breaks:** Clients retry on timeout, mobile users double-tap, and every payment provider and
queue delivers at least once. Without idempotency, one intent becomes two charges.

**Fix:** Accept an `Idempotency-Key` header on unsafe operations, store it with the result, and
return the stored result on repeat. For webhooks, insert the provider's event id into a table with a
unique constraint and skip the work if the insert conflicts. For jobs, make the handler check
whether the effect already exists before performing it.

---

## Connection pool exhaustion

**Signal:** A new client/connection created inside a request handler or per function call. No pool
configuration. On serverless, a pool sized for a long-lived server.

```js
app.get('/x', async (req, res) => {
  const client = new Client(url);          // new connection every request, never closed
  await client.connect();
});
```

**Why it breaks:** The database hits `max_connections` and every request starts failing at once,
including health checks. On serverless the failure is worse: each concurrent invocation may hold its
own connections.

**Fix:** One pool, created at module scope, reused. Size it to `max_connections` divided by the
number of instances, with headroom. On serverless, use a connection pooler in front of the database
(PgBouncer, Prisma Accelerate, Neon/Supabase pooled endpoints, RDS Proxy) and keep the per-instance
pool small (1–5). Always release connections in a `finally`.

**Also check:** every acquired connection, cursor, file handle, and HTTP agent is released on the
error path, not just the happy path.

---

## N+1 queries

**Signal:** A loop containing a query, or a serializer that accesses a relation per item.

```js
const posts = await Post.findAll();
for (const p of posts) p.author = await User.findById(p.authorId);   // 1 + N
```

**Why it breaks:** 50 rows becomes 51 round trips. Fine on a laptop with 10 rows of seed data, fatal
at production volume — this is the most common reason an AI-generated backend is slow.

**Fix:** Eager-load the relation: `include`, `select_related`/`prefetch_related`, `with()`,
`includes()`, `JOIN FETCH`, `Include()`. Or fetch ids in one pass and load with a single
`WHERE id IN (...)`, then map in memory. Enable the ORM's query logging in development and count
queries per request — the count should not scale with row count.

---

## Missing indexes

**Signal:** Columns used in `WHERE`, `JOIN`, `ORDER BY`, or foreign keys with no index. Look at the
migration files, not the model definitions — some ORMs do not create an index for a foreign key
automatically.

**Why it breaks:** Full table scans. The app is fast until the table grows, then degrades sharply and
takes the whole database's CPU with it.

**Fix:** Index every foreign key, every column filtered in a hot query, and use composite indexes
matching the query's filter order for multi-column filters. Verify with `EXPLAIN`. Do not index
everything — each index slows writes.

**Also check:** unique constraints exist where uniqueness is assumed (`email`, `slug`, `(tenant_id,
name)`), and that they are enforced at the database level rather than only in application code.

---

## Unbounded queries

**Signal:** `findAll()`, `.objects.all()`, `SELECT *` with no `LIMIT`, or an endpoint that returns
every row.

**Why it breaks:** Works with 100 rows, exhausts memory at 1,000,000. One request can take the
process down.

**Fix:** Paginate by default with a maximum page size the caller cannot exceed. Prefer cursor
pagination over `OFFSET` for large tables — deep offsets get slower linearly. Stream when a full
export is genuinely required.

---

## Schema and constraints

**Signal:** Everything nullable. No foreign keys. Enums stored as free text. Money in `float`.
Validation only in application code.

**Why it breaks:** The database is the last line of defense. Application validation is bypassed by
migrations, scripts, admin tools, background jobs, and the next service you write. Nullable-everything
means every read site needs a null check, and one will be missing.

**Fix:** `NOT NULL` wherever a value is required, foreign keys with a deliberate `ON DELETE`
behavior, `CHECK` constraints for bounded values, and unique constraints for uniqueness. Add
constraints even to an existing database — validating them will reveal the corrupt rows you already
have.

---

## Migrations

**Signal:** Schema changed by hand or by `synchronize: true` / `db.create_all()` in production.
Migrations that drop or rename a column in the same deploy that changes the code. No rollback path.

**Why it breaks:** Auto-sync can drop columns to match the model. A rename during a rolling deploy
means old instances query a column that no longer exists — a hard outage for the duration.

**Fix:** Migrations in version control, applied by an explicit step. Never auto-sync in production.
Make changes expand-then-contract: add the new column, backfill, write both, switch reads, stop
writing the old one, drop it in a later deploy. Test the rollback. On large tables, add indexes
concurrently and avoid rewriting locks.

---

## Soft deletes

**Signal:** A `deleted_at`/`is_deleted` column, with some queries filtering on it and some not.

**Why it breaks:** Deleted records reappear in lists, search, exports, and counts. Worse, a unique
constraint on `email` blocks re-registration after a "deletion" — and a soft-deleted user may still
authenticate if the login query does not filter.

**Fix:** Apply the filter in a default scope so it cannot be forgotten, and audit every raw query for
it. Check specifically that authentication, authorization, and billing paths exclude soft-deleted
rows. Make unique constraints partial (`WHERE deleted_at IS NULL`) where re-use should be allowed.

---

## Timezones and money

**Signal:** `DATETIME` without timezone, `NOW()` in application code with a local timezone,
`float`/`double`/`REAL` for currency, prices as strings parsed ad hoc.

**Why it breaks:** Timestamps shift by hours between environments; "today" differs per server.
Floats cannot represent 0.1 — totals drift by cents and reconciliation fails.

**Fix:** Store timestamps as UTC with timezone (`timestamptz`), convert only at the presentation
layer. Store money as integer minor units (cents) or `DECIMAL/NUMERIC` with explicit precision,
never float. Store the currency code alongside the amount.

---

## Backups and recovery

**Signal:** No backup configuration, or backups that have never been restored.

**Why it breaks:** An untested backup is a hypothesis. The common failure is discovering during an
incident that the backup excludes a schema, or that restore takes eight hours.

**Fix:** Automated backups with a stated retention, point-in-time recovery for anything
transactional, and at least one documented restore into a scratch environment. Know the RPO and RTO
and write them down. Verify destructive operations require a deliberate step — no `DROP`/`TRUNCATE`
reachable from application code, and no production credentials in a local `.env`.
