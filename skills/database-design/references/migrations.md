# Migrations

The most dangerous file in the repository. Load when reviewing a migrations folder, or before
proposing any schema change.

**Never propose a migration without saying whether it locks the table and for how long.** This is the
one area where the failure mode is downtime or data loss rather than slowness.

Two rules govern almost everything below:

> **Additive changes:** migrate first, then deploy.
> **Destructive changes** (DROP, RENAME, narrowing a type, SET NOT NULL): deploy first, then migrate.

Every entry has a **Fine when** line. Read it before reporting — a pre-launch project with no users
should not be told to run a six-step rollout.

## Contents

**Postgres locking** — `ADD COLUMN` with a volatile default · `SET NOT NULL` · `CREATE INDEX` without
`CONCURRENTLY` · missing `lock_timeout` · `ALTER COLUMN TYPE` · adding a foreign key or check ·
`ALTER TYPE ADD VALUE`
**MySQL** — `ALTER TABLE` without `ALGORITHM` · multi-statement migrations
**SQLite** — the 12-step rebuild
**MongoDB** — index builds
**Any engine** — renames and drops · unbatched backfills · editing an applied migration

---

## Postgres: `ADD COLUMN` with a volatile default

**Signal:** `ALTER TABLE t ADD COLUMN c uuid NOT NULL DEFAULT gen_random_uuid()`.

**Why it breaks:** On PG < 11 *any* default rewrote the entire table and all its indexes under
`ACCESS EXCLUSIVE`. On PG 11+ a non-volatile default is metadata-only, but a **volatile** one
(`gen_random_uuid`, `uuid_generate_v4`, `random`, `clock_timestamp`), a `STORED` generated column, an
identity column, or a domain type with constraints still rewrites every row. On 50M rows that is
minutes with all readers and writers blocked.

**Fix:**

```sql
ALTER TABLE users ADD COLUMN token uuid;                            -- instant
-- backfill in batches (see below)
ALTER TABLE users ALTER COLUMN token SET DEFAULT gen_random_uuid();
-- then SET NOT NULL via the CHECK route
```

**Fine when:** the default is `now()` or a literal. Those are stable or immutable and do **not** rewrite
on PG 11+ — this is the most common false positive in this whole file. Also fine at any version on a
table created in the same migration.

---

## Postgres: `SET NOT NULL`

**Signal:** `ALTER TABLE t ALTER COLUMN c SET NOT NULL`, Django `AlterField null=False`, Prisma
removing `?`, Rails `change_column_null`.

**Why it breaks:** Postgres holds `ACCESS EXCLUSIVE` while it scans every row to verify. There is no
rewrite, but on a large table the scan takes minutes and **every SELECT queues behind the lock** for
the whole time.

**Fix (PG 12–17):**

```sql
ALTER TABLE t ADD CONSTRAINT t_c_nn CHECK (c IS NOT NULL) NOT VALID;  -- instant
ALTER TABLE t VALIDATE CONSTRAINT t_c_nn;    -- SHARE UPDATE EXCLUSIVE only
ALTER TABLE t ALTER COLUMN c SET NOT NULL;   -- scan skipped, lock held for ms
ALTER TABLE t DROP CONSTRAINT t_c_nn;
```

PG 18+ accepts `NOT VALID` on the `NOT NULL` constraint directly.

**Fine when:** under a few hundred thousand rows, or on a table created in the same migration.

The skip-the-scan optimisation **requires PG 12+**. Below that the scan is unavoidable and the only
safe route is a maintenance window.

---

## Postgres: `CREATE INDEX` without `CONCURRENTLY`

**Signal:** `CREATE INDEX` with no `CONCURRENTLY` — or `CREATE INDEX CONCURRENTLY` in a migration file
with no transaction opt-out.

**Why it breaks:** Non-concurrent `CREATE INDEX` takes `SHARE` and blocks all writes for the whole
build. But `CONCURRENTLY` **cannot run inside a transaction block**, and Rails, Django, Alembic, and
Prisma all wrap each migration in one — so it errors with `25001`, or worse, a failed concurrent build
leaves an **INVALID** index that queries ignore while every write still maintains it.

**Fix:** put it alone in its own migration, with the transaction disabled.

```sql
CREATE INDEX CONCURRENTLY idx ON t (col);
```

```ruby
disable_ddl_transaction!                                  # Rails
add_index :t, :col, algorithm: :concurrently
```

```python
atomic = False                                            # Django
django.contrib.postgres.operations.AddIndexConcurrently
with op.get_context().autocommit_block():                 # Alembic
    op.create_index(..., postgresql_concurrently=True)
```

**Prisma has no per-migration transaction opt-out** — the workaround is one statement per migration
file.

**Fine when:** the table is created in the same migration, or is under ~100k rows. `CONCURRENTLY` does
two table scans and is 2–3× slower.

**Find leftovers:** `SELECT c.relname FROM pg_index i JOIN pg_class c ON c.oid = i.indexrelid WHERE
NOT i.indisvalid;`

---

## Postgres: missing `lock_timeout`

**Signal:** no `SET lock_timeout` anywhere in the migrations directory, the runner config, or the
deploy script. **Absence is the finding.**

**Why it breaks:** Postgres grants locks FIFO. An `ALTER TABLE` waiting for `ACCESS EXCLUSIVE` behind
one long-running `SELECT` blocks **every subsequent reader** of that table. A metadata-only DDL that
takes 1ms can still cause a full outage lasting as long as the oldest query.

**Fix:**

```sql
SET lock_timeout = '3s';    -- fail fast instead of queueing
ALTER TABLE t ...;
```

On SQLSTATE `55P03` (`lock_not_available`), sleep and retry the migration.

**Check for blockers before running:**

```sql
SELECT pid, state, now() - xact_start AS age, query FROM pg_stat_activity
 WHERE state <> 'idle' ORDER BY age DESC LIMIT 5;
```

**Fine when:** the statement is `CREATE INDEX CONCURRENTLY` or `VALIDATE CONSTRAINT` —
`SHARE UPDATE EXCLUSIVE` does not conflict with reads.

**Only report this together with the retry loop.** A short `lock_timeout` with no retry just converts
an outage into a failed deploy.

MySQL's equivalent `lock_wait_timeout` defaults to 31536000 seconds — one year, effectively no timeout.

---

## Postgres: `ALTER COLUMN TYPE`

**Signal:** `ALTER TABLE t ALTER COLUMN c TYPE ...`, Prisma `Int` → `BigInt`, Django `AlterField`
changing the field class.

**Why it breaks:** Full table and index rewrite under `ACCESS EXCLUSIVE` unless the old type is
binary-coercible to the new one. `int` → `bigint`, numeric precision changes, and shrinking a varchar
all rewrite. The table is unreadable for the entire rewrite and needs disk space for a second copy.

**Fix:** these are free and should not be flagged —

```
varchar(50) -> varchar(200)      varchar(n) -> text      text -> varchar
```

Anything that rewrites needs expand-and-contract: add a new column, backfill in batches, dual-write,
swap.

**Fine when:** widening a varchar or converting to text.

Shrinking a varchar **does** scan and rewrite. `timestamp` → `timestamptz` avoids the rewrite on PG 12+
only when the session `TimeZone` is UTC. Index rebuilds are skipped for text↔varchar but forced by any
collation change.

---

## Postgres: adding a foreign key or check constraint

**Signal:** `ADD CONSTRAINT ... FOREIGN KEY` or `ADD CHECK` with no `NOT VALID`. Rails
`add_foreign_key` without `validate: false`.

**Why it breaks:** Adding a foreign key takes `SHARE ROW EXCLUSIVE` on **both** the child and the
parent table — all writes to both are blocked — while it scans the child. So adding an FK on a tiny new
table blocks writes on the huge table it points at. That is the part nobody expects.

**Fix:**

```sql
ALTER TABLE orders ADD CONSTRAINT orders_user_fk
  FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;   -- instant
ALTER TABLE orders VALIDATE CONSTRAINT orders_user_fk;    -- SHARE UPDATE EXCLUSIVE
```

**Fine when:** the table is under ~100k rows, or the constraint is part of a table created in the same
migration.

`NOT VALID` still enforces the constraint on all new and modified rows — only pre-existing rows go
unchecked until `VALIDATE`.

---

## Postgres: `ALTER TYPE ADD VALUE`

**Signal:** `ALTER TYPE status ADD VALUE 'refunded'`, or an ORM-generated enum change emitting
`DROP TYPE` + `CREATE TYPE`.

**Why it breaks:** Inside a transaction block the new value **cannot be used until after commit**, so a
migration that adds a value and then `UPDATE`s rows to use it fails. On PG < 12 the statement is a hard
error inside a transaction at all. Tools that regenerate the enum (Prisma, TypeORM) emit
`DROP TYPE`/`CREATE TYPE`, which rewrites every table using it under `ACCESS EXCLUSIVE`.

**Fix:** own migration file, transaction disabled, and any `UPDATE` using the new value goes in a
**later** migration.

```sql
ALTER TYPE order_status ADD VALUE IF NOT EXISTS 'refunded';
```

**Fine when:** adding at the end of the list — that is cheap. Using `BEFORE`/`AFTER` to position a value
makes later comparisons slower.

Enum values can never be removed or reordered. If the set is expected to churn, a text column plus a
`CHECK`, or a lookup table, is the safer design — but converting an existing enum is itself a rewriting
migration, so weigh that before recommending it.

---

## MySQL: `ALTER TABLE` without `ALGORITHM`

**Signal:** any `ALTER TABLE` in a MySQL migration with no `, ALGORITHM=..., LOCK=...` clause.

**Why it breaks:** MySQL silently chooses the algorithm. Column type changes, shrinking a VARCHAR,
widening a VARCHAR across the 255-byte length-prefix boundary (63 → 64 characters in utf8mb4),
`DROP PRIMARY KEY`, adding a `STORED` generated column, and most partitioning changes all fall back to
`ALGORITHM=COPY`, which blocks writes for the full copy.

**Stating the algorithm makes MySQL raise `ERROR 1845` instead of quietly locking your table for an
hour.**

**Fix:**

```sql
ALTER TABLE orders ADD COLUMN note TEXT, ALGORITHM=INSTANT;
ALTER TABLE orders ADD INDEX idx_x (x), ALGORITHM=INPLACE, LOCK=NONE;
```

If it errors, the change is not online — run it through a copy tool:

```bash
gh-ost --alter="ADD COLUMN note TEXT" --table=orders --execute
pt-online-schema-change --alter "ADD COLUMN note TEXT" D=db,t=orders --execute
```

**Fine when:** under a few million rows, where neither copy tool is worth its complexity.

Version limits: `INSTANT ADD COLUMN` needs 8.0.12+ (any position only from 8.0.29; instant
`DROP COLUMN` from 8.0.29). MySQL 5.7 has no `INSTANT` at all. Instant add/drop is capped at **64 row
versions per table** — past that you get `ERROR 4092` and must rebuild. Even `INPLACE` takes a brief
exclusive metadata lock at start and commit, so it still queues behind a long transaction.

`gh-ost` refuses tables that are FK parents or children, or that already have triggers, and needs
`binlog_format=ROW`. `pt-osc` handles FKs but adds triggers to the live table.

---

## MySQL: multi-statement migrations

**Signal:** a MySQL migration file containing several DDL statements.

**Why it breaks:** **Every DDL statement in MySQL implicitly commits.** If statement 3 of 5 fails,
statements 1 and 2 stay applied while the tool records the migration as failed. The schema is now in a
state no migration file describes, and re-running dies on "column already exists."

**Fix:** one DDL statement per migration file, each idempotent.

```sql
ALTER TABLE orders ADD COLUMN IF NOT EXISTS note TEXT;
DROP INDEX IF EXISTS idx_old ON orders;
```

**Fine when:** the engine is Postgres or SQLite. Both have transactional DDL, so multi-statement
migrations are perfectly safe there — **do not flag them.**

MySQL 8.0's atomic DDL guarantees a *single* statement will not half-apply; the gap is only across
statements.

---

## SQLite: the 12-step rebuild

**Signal:** a migration against SQLite that drops a column, changes a type, adds or removes a
constraint, or adds a `UNIQUE` column.

**Why it breaks:** SQLite has no `ALTER` for those, so the ORM silently performs the 12-step rebuild:
create a new table, copy every row, drop the original, rename. It is O(table size), and if
`PRAGMA foreign_keys` is ON during the rebuild, the `DROP TABLE` step can cascade-delete rows in child
tables or leave references pointing at the temporary name.

**Fix:**

```sql
PRAGMA foreign_keys=OFF;    -- must be OUTSIDE any transaction; a no-op inside one
BEGIN;
CREATE TABLE new_t (...);
INSERT INTO new_t SELECT ... FROM t;
DROP TABLE t;
ALTER TABLE new_t RENAME TO t;
-- recreate every index, trigger and view
PRAGMA foreign_key_check;
COMMIT;
PRAGMA foreign_keys=ON;
```

**Fine when:** the operation has a native cheap `ALTER` — `RENAME TABLE`, `ADD COLUMN`, `RENAME COLUMN`
(3.25.0+), `DROP COLUMN` (3.35.0+), `SET`/`DROP NOT NULL` (3.53.0+). Do not flag those. Also
irrelevant on a single-user development database.

`ADD COLUMN` legitimately cannot carry `UNIQUE`/`PRIMARY KEY`, a `CURRENT_TIMESTAMP` or
parenthesised-expression default, or `NOT NULL` without a non-null constant default. Those errors are
SQLite working as designed, not a defect to fix.

---

## MongoDB: index builds

**Signal:** `createIndex(..., { background: true })`, `ensureIndex`, index creation inside application
startup or a request handler, or a Mongoose connection with `autoIndex` at its default.

**Why it breaks:** `background` **has had no effect since MongoDB 4.2** — all builds use the hybrid
build, which takes an exclusive collection lock at the start and at commit and a shared lock during the
drain phase, so writes do block at the boundaries. Mongoose `autoIndex` means every process restart
re-issues index builds against production.

**Fix:**

```js
mongoose.connect(uri, { autoIndex: false });
// then build deliberately, from a one-off script or migration
db.orders.createIndex({ userId: 1, createdAt: -1 });
```

**Fine when:** the collection is small — the build is milliseconds either way. `background: true` is
still accepted for backward compatibility and is dead weight, not a bug.

Rolling index builds only pay off on very large collections; they require an election and reduce
redundancy while running. A unique index build **aborts outright** if duplicates already exist.

---

## Any engine: renames and drops

**Signal:** `RENAME COLUMN`, `DROP COLUMN`, a dropped table, or an ORM field rename, as a single
migration with no add-then-remove sequence.

**Why it breaks:** The migration and the code deploy are never simultaneous. A rename breaks the
currently-running old code the instant it applies, and breaks the new code for the entire window before
it applies. **There is no ordering that works.** Rollback is impossible once the old column is gone.

**Fix — expand and contract:**

```
1  migrate: ADD new column, nullable          (migrate first, then deploy)
2  deploy:  code writes BOTH columns, reads old
3  backfill in batches
4  deploy:  code reads new, still writes both
5  deploy:  code stops touching old
6  migrate: DROP old column                   (deploy first, then migrate)
```

**Fine when:** the column was added earlier in the same unreleased branch, or the table has no
production traffic yet. Expand-and-contract only earns its cost when old and new code run at the same
time. **Do not tell a pre-launch project to do six steps.**

**Find it:** grep the application code for the old column name to see whether anything still reads it.

---

## Any engine: unbatched backfills

**Signal:** `UPDATE <table> SET col = ...` with no bounded `WHERE` inside a migration, or a Django
`RunPython` calling `.update()` on a full queryset.

**Why it breaks:** One transaction touching every row holds row locks across the whole table, doubles
the table's size in dead tuples on Postgres (autovacuum cannot reclaim rows the open transaction still
needs), and stalls replicas. If it hits a statement timeout halfway, everything rolls back and nothing
was accomplished.

**Fix:** loop **outside** the migration transaction, committing each batch.

```sql
UPDATE t SET col = <expr>
WHERE id IN (SELECT id FROM t WHERE col IS NULL ORDER BY id LIMIT 5000);
-- repeat until 0 rows affected; sleep briefly between batches
```

**Fine when:** under roughly 100k rows, or at any size on a table production does not read yet.

Batching **needs an index on the filter column** — otherwise every batch does a full sequential scan and
the loop is slower than the one-shot update.

---

## Any engine: editing an applied migration

**Signal:** a migration file modified in a commit later than the one that created it. Or a
`down`/`downgrade` body that is empty, `pass`, or `raise`.

**Why it breaks:** Flyway and Liquibase store a checksum and refuse to run until `flyway repair`.
Django, Rails, and Prisma do **not** checksum, so the edit silently applies only to databases created
from scratch — production keeps the old schema forever while dev and CI diverge invisibly. Prisma
`migrate dev` responds to the resulting drift by offering to reset the database, **which drops all
data**.

**Fix:** never edit an applied migration; add a new one.

```python
def downgrade():
    raise NotImplementedError("drops data; recover from backup")
```

**Fine when:** the migration has only ever run on your own machine and has not been merged or deployed.
Squashing local migrations before opening a PR is good practice.

A truly reversible `down` for `DROP COLUMN` does not exist. Documenting the restore path is the honest
answer, not a fabricated reversal.

**Find it:** `git log --diff-filter=M --name-only -- 'migrations/*'` — any file listed was edited after
creation.

---

## Sources

PostgreSQL: [sql-altertable](https://www.postgresql.org/docs/current/sql-altertable.html),
[sql-createindex](https://www.postgresql.org/docs/current/sql-createindex.html),
[sql-altertype](https://www.postgresql.org/docs/current/sql-altertype.html).
MySQL: [innodb-online-ddl-operations](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html),
[innodb-online-ddl-performance](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-performance.html).
SQLite: [lang_altertable](https://www.sqlite.org/lang_altertable.html).
MongoDB: [index-creation](https://www.mongodb.com/docs/manual/core/index-creation/).
Also [gh-ost requirements](https://github.com/github/gh-ost/blob/master/doc/requirements-and-limitations.md)
and [squawk](https://squawkhq.com/docs/ban-concurrent-index-creation-in-transaction).
