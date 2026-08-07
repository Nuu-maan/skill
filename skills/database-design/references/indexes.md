# Indexes

Index design across all four engines. Load when reviewing any index, or any query that filters,
joins, or sorts.

Two rules underlie most of this file:

1. **An index that exists is not an index that is used.** Most findings here are indexes the planner
   cannot choose — wrong column order, a function on the column, a predicate that does not match.
2. **Every index costs writes, forever.** Never recommend one without a query that needs it.

Every entry has a **Fine when** line. Read it before reporting.

## Contents

- Unindexed foreign keys in Postgres
- A function on the column disables the index
- Composite column order
- MongoDB ESR: equality, sort, range
- Redundant prefix indexes
- Payload columns in the key instead of `INCLUDE`
- Partial indexes whose predicate is never matched
- Choosing the index type
- MongoDB sparse indexes silently return wrong results
- MongoDB TTL indexes
- `CREATE INDEX` without `CONCURRENTLY`
- Too many indexes
- SQLite uses one index per table per query

---

## Unindexed foreign keys in Postgres

**Signal:** A `REFERENCES` clause (or Prisma `@relation`, Drizzle `references()`) with no matching
`CREATE INDEX` / `@@index` on the **child-side** column.

**Why it breaks:** Postgres indexes the parent's referenced key automatically but never the child
column. Every `DELETE` or key `UPDATE` on the parent sequentially scans the whole child table once per
affected row to check for referencing rows — while holding a row lock. Deleting one user with five
unindexed child tables is five full table scans. At 10M rows this turns a 1ms delete into minutes and
blocks concurrent writers.

**Fix:**

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);
```

```prisma
model Order {
  userId String
  user   User @relation(fields: [userId], references: [id])
  @@index([userId])
}
```

**Fine when:** the child column is already the leftmost column of another index, the parent table is
immutable reference data never deleted from or re-keyed, or the child table is small (under ~10k rows)
and stays that way. An index costs write throughput — do not add one to a tiny lookup table.

**Engine note:** this applies to Postgres and SQLite. **MySQL/InnoDB auto-creates a child-side index**
if none exists with the FK columns leftmost, so it is not a finding there.

> Prisma's own documentation claims "the database usually also implicitly creates an index for the
> foreign key columns." That is **false for Postgres**, even in `relationMode = "foreignKeys"`. Do not
> trust it.

---

## A function on the column disables the index

**Signal:** A column wrapped in a function inside `WHERE` or `JOIN` — `WHERE lower(email) = $1`,
`WHERE DATE(created_at) = ?`, `WHERE CAST(id AS text) = ...`, `WHERE user_id::text = ...`. In ORMs:
`func.lower(User.email) ==`.

**Why it breaks:** A plain btree index on `email` sorts raw values, so `lower(email)` is opaque to the
planner and the query falls back to a sequential scan. This is the single most common reason an index
that "exists" is never used, and it usually appears on login and lookup paths that run on every
request.

**Fix:**

```sql
CREATE INDEX ON users ((lower(email)));           -- Postgres, function must be IMMUTABLE
ALTER TABLE users ADD INDEX ((lower(email)));     -- MySQL 8.0.13+
CREATE INDEX ON users (lower(email));             -- SQLite 3.9.0+
```

Or store a normalised column and index that.

**Fine when:** the expression is on the *right* side (`email = lower($1)`), the table is small, or the
column is already stored normalised.

MySQL before 8.0.13 has no expression indexes at all — use a generated column. Postgres will not
accept a non-`IMMUTABLE` function (`to_char` with a timezone, `now()`), so an expression index is not
always available; a generated column is the fallback.

**Find it:** `rg -ni "where[^;]*\b(lower|upper|date|cast|coalesce|to_char|trim|substr)\s*\("`. Also
flag `::` casts on a column to the left of a comparison.

---

## Composite column order

**Signal:** A composite index whose column order does not match the query's predicate shape — e.g.
`INDEX (created_at, tenant_id)` while every query is `WHERE tenant_id = ? ORDER BY created_at DESC`.

**Why it breaks:** Btree indexes are only narrowed by equality constraints on the leading columns, plus
one inequality on the first unconstrained column. Put the range column first and the equality column is
only *filtered*, not seeked — the engine reads a huge slice of the index. Wrong order can be the
difference between reading 10 index entries and 10 million.

**Fix:** equality columns first, then the sort or range column.

```sql
CREATE INDEX ON events (tenant_id, created_at DESC);
-- serves  WHERE tenant_id = ? ORDER BY created_at DESC
-- and     WHERE tenant_id = ? AND created_at > ?
```

**Fine when:** the leading column genuinely carries an equality predicate in the real workload, or the
index exists to serve a different query than the one you are reading.

Postgres 18 added btree skip scan, which partially rescues a missing leading equality when that column
has few distinct values — so on PG 18+ this is a slowdown, not necessarily a table scan.

**Find it:** list every composite index, then compare its first column against the equality predicates
in the hottest queries. A leading column that never appears with `=` in any query is the tell.

---

## MongoDB ESR: equality, sort, range

**Signal:** Range operators (`$gt`, `$lt`, `$ne`, `$nin`, `$regex`) placed before the `sort()` field in
a compound index key.

**Why it breaks:** An index supports a sort only when all index keys preceding the sort keys have
equality conditions. A range key before the sort key breaks that, so MongoDB runs a blocking in-memory
`SORT` stage — which **fails outright at 100 MB** unless `allowDiskUse` is on. Under load this appears
as sudden query failures rather than gradual slowness.

**Fix:**

```js
// query: find({ directors: "X", runtime: { $lt: 130 } }).sort({ year: 1 })
db.movies.createIndex({ directors: 1, year: 1, runtime: 1 })
//                      Equality      Sort     Range
```

**Fine when:** the range predicate is far more selective than the sort — then E-R-S can win, because it
sorts far fewer documents. ESR is a guideline, not a law.

`$in` behaves as equality only below 201 array elements. At 201+ it is ordered like a range and blocks
index sorting on later fields, and MongoDB states that threshold may change between versions.

**Confirm it:** `db.coll.find(...).sort(...).explain()` — a `SORT` stage in `winningPlan` means the
index is not providing the sort.

---

## Redundant prefix indexes

**Signal:** Two indexes where one key list is a leading prefix of the other — `INDEX (user_id)`
alongside `INDEX (user_id, created_at)`.

**Why it breaks:** The narrower index is dead weight: the wider one serves every query it does. You pay
its full write and storage cost on every `INSERT`/`UPDATE`/`DELETE` forever and gain nothing.

On InnoDB it is worse than it looks — **every secondary index silently has the primary key columns
appended**, so `INDEX (d)` on a table with `PRIMARY KEY (i1, i2)` is physically `(d, i1, i2)`, which
makes an explicit `(d, i1)` index redundant too.

**Fix:**

```sql
DROP INDEX idx_orders_user_id ON orders;          -- covered by (user_id, created_at)
ALTER TABLE orders ALTER INDEX idx_x INVISIBLE;   -- MySQL 8.0+, reversible test first
```

**Fine when:** the narrow index is `UNIQUE` — a unique constraint on `(a)` is **not** implied by an
index on `(a, b)` — or it differs in partial predicate, opclass, or sort direction.

Dropping a unique index because it "looks like a prefix" is a data-integrity bug, not an optimisation.

**Find it:** MySQL has this built in: `SELECT * FROM sys.schema_redundant_indexes;`

---

## Payload columns in the key instead of `INCLUDE`

**Signal:** An index listing a column that is only ever SELECTed, never filtered or sorted on —
`CREATE INDEX ON orders (status, total_cents)` where `total_cents` never appears in a `WHERE` or
`ORDER BY`.

**Why it breaks:** Postgres `INCLUDE` stores the payload in leaf pages only, not in internal nodes, so
the index stays smaller and stays sorted purely on the key. Putting a payload column in the key bloats
every level of the tree and — on a `UNIQUE` index — silently changes the uniqueness semantics from
`(status)` to `(status, total_cents)`.

**Fix:**

```sql
CREATE INDEX ON orders (status) INCLUDE (total_cents);
CREATE UNIQUE INDEX ON orders (order_no) INCLUDE (total_cents);   -- UNIQUE keeps its meaning
```

**Fine when:** the index-only scan would not fire anyway. Postgres index-only scans require heap pages
marked all-visible in the visibility map, so on a heavily-updated table that autovacuum cannot keep up
with, you pay the extra index size and still hit the heap.

Expressions are not allowed as `INCLUDE` columns, and an oversized index tuple makes `INSERT` fail
outright. `INCLUDE` is Postgres 11+ (btree; GiST and SP-GiST from 14). **MySQL, SQLite, and MongoDB
have no `INCLUDE`** — for them, adding trailing key columns *is* the covering technique.

Prisma and Drizzle cannot express `INCLUDE`; it needs raw SQL in the migration.

---

## Partial indexes whose predicate is never matched

**Signal:** A partial index whose predicate does not appear verbatim in the queries —
`CREATE INDEX ... WHERE deleted_at IS NULL` while the code queries
`WHERE deleted_at IS NULL OR deleted_at > now()`. In MongoDB, a `partialFilterExpression` whose
condition is missing from the query document.

**Why it breaks:** Postgres only uses a partial index when it can prove the predicate is implied by the
`WHERE` clause, and it does not reason about mathematically equivalent rewrites. MongoDB will not use a
partial index unless the query predicate matches or is a subset of the filter expression. So the index
exists, costs writes, and is never chosen — the worst of both worlds.

**Fix:** make the application query state the predicate literally.

```sql
CREATE INDEX ON orders (user_id) WHERE deleted_at IS NULL;
-- the query must contain:  AND deleted_at IS NULL
```

```js
find({ name: "x", password: { $exists: true } })   // repeats the filter expression
```

**Fine when:** the predicate *is* repeated in queries. A unique partial index is the standard way to
enforce "one active row per user":

```sql
CREATE UNIQUE INDEX ON subs (user_id) WHERE status = 'active';
```

Do not build many non-overlapping partial indexes as a substitute for partitioning; one composite index
on the category column is better.

> **MySQL has no partial or filtered indexes at all.** `CREATE INDEX ... WHERE` is a syntax error, and
> AI-written MySQL schemas hallucinate it regularly. On MySQL this is an outright migration failure,
> not a performance note.

---

## Choosing the index type

**Signal:** `USING hash`, `USING gin`, `USING brin` in a migration. Or `LIKE '%term%'` / `ILIKE`
queries against a plain btree index. Or a `text` column searched with `to_tsvector` and no GIN index.

**Why it breaks:** Each type has a hard limitation the author often did not check. Hash supports only
`=` — no ordering, no range, no multicolumn, no `UNIQUE`. BRIN is useless unless the column's values
correlate with physical row order, and degrades silently to near-full scans on random data. GIN has
expensive updates. Btree cannot serve a leading-wildcard `LIKE` at all.

**Fix:**

```sql
-- substring search
CREATE EXTENSION pg_trgm;
CREATE INDEX ON docs USING gin (body gin_trgm_ops);

-- full text
CREATE INDEX ON docs USING gin (to_tsvector('english', body));

-- append-only time series only
CREATE INDEX ON events USING brin (created_at);
```

**Fine when:** used in its niche. BRIN on an append-only `created_at` over 100M rows is a tiny index
that beats btree on cost. GIN is correct for jsonb containment and arrays. Hash is narrower than btree
and rarely worth it since PG 10 made it crash-safe — limited, not broken.

Do not flag any of these without checking the query shape first.

**Engine note:** MySQL InnoDB supports only BTREE, FULLTEXT, and SPATIAL — `USING HASH` is silently
accepted and ignored. SQLite has btree only. MongoDB has text, 2dsphere, and wildcard.

---

## MongoDB sparse indexes silently return wrong results

**Signal:** `{ sparse: true }`, especially combined with a `.sort()` on that field, or used as an
attempt at conditional uniqueness.

**Why it breaks:** MongoDB refuses to use a sparse index for any query or sort that would return
incomplete results, so an unqualified `find().sort({ plot: -1 })` silently ignores it and does a
collection scan. If you force it with `hint()`, documents missing the field **vanish from results** and
`countDocuments({}, { hint: ... })` returns a wrong number — a correctness bug, not a performance one.

**Fix:**

```js
db.movies.createIndex(
  { plot: 1 },
  { partialFilterExpression: { plot: { $exists: true } } }
)
```

**Fine when:** `{ sparse: true, unique: true }` for optional-unique fields — allow many documents
missing the field, unique when present. Partial with `$exists: true` is the modern equivalent, but
sparse is not wrong.

2d, 2dsphere v2, text, and wildcard indexes are always sparse whether you ask or not. Sparse and
`partialFilterExpression` cannot be combined.

---

## MongoDB TTL indexes

**Signal:** `expireAfterSeconds` on a compound key, on a non-date field, on `_id`, or a value the code
later tries to change by calling `createIndex` again.

**Why it breaks:** TTL indexes are single-field only and require a BSON date (or an array of dates —
the earliest wins). A TTL index on a number or an ISO **string** silently never deletes anything, so
the collection grows unbounded until the disk fills. Documents missing the field never expire either.

**Fix:**

```js
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })

// changing the TTL later — createIndex will NOT update it
db.runCommand({ collMod: "sessions",
  index: { keyPattern: { expiresAt: 1 }, expireAfterSeconds: 300 } })
```

**Fine when:** you accept that deletion is eventual. A background thread runs every 60 seconds on the
primary only, deleting up to 50,000 documents or one second of work per index per pass. Backlogs on a
large collection are normal, not a bug.

**Never treat TTL as a security or correctness boundary.** Always also filter `expiresAt > now` in the
query.

**Find it:** verify the field is written as a `Date`, not `Date.now()` (a number) or `.toISOString()`
(a string).

---

## `CREATE INDEX` without `CONCURRENTLY`

**Signal:** A plain `CREATE INDEX` in a migration against a table that already has production data —
especially inside a framework that wraps migrations in a transaction (Prisma, Rails, Alembic, Drizzle).

**Why it breaks:** A plain Postgres `CREATE INDEX` blocks all `INSERT`/`UPDATE`/`DELETE` on the table
for the whole build. Reads continue; writes queue. On a multi-million-row table that is minutes of
write outage during deploy — the classic "the deploy took the site down" incident.

**Fix:**

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);
```

Rails: `disable_ddl_transaction!` plus `algorithm: :concurrently`. Prisma: a manually edited
`migration.sql` — `CONCURRENTLY` must not run inside a transaction.

**Fine when:** the table is brand new or empty, it is an initial-schema migration, or write traffic is
negligible. There, plain `CREATE INDEX` is faster and safer.

`CONCURRENTLY` does two table scans, takes considerably longer, cannot run inside a transaction block,
is unsupported on partitioned tables, and on failure leaves an **INVALID** index you must drop and
rebuild:

```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
```

MySQL 5.6+ InnoDB already does most `ADD INDEX` online. SQLite locks the whole database file briefly.

---

## Too many indexes

**Signal:** More indexes than the table has distinct query shapes — commonly one index per column,
which AI-generated schemas produce almost by reflex (`@@index` on every field, ten `CREATE INDEX` lines
on a six-column table).

**Why it breaks:** Every index must be updated on every `INSERT` and on every `UPDATE` touching its
columns, each write is WAL/redo-logged, and each competes for buffer cache. Ten indexes on a
write-heavy table can cut insert throughput several-fold. Single-column indexes are usually also the
wrong *shape* — one well-ordered composite typically replaces three.

**Fix:**

```sql
DROP INDEX idx_tenant; DROP INDEX idx_status; DROP INDEX idx_created;
CREATE INDEX ON events (tenant_id, status, created_at DESC);
```

**Fine when:** the table is read-mostly — reference data, analytics, materialised reporting tables. The
write cost is what matters, not the count.

MongoDB caps at 64 indexes per collection and 32 fields per compound index.

**Confirm unused indexes at runtime:**

```sql
-- Postgres
SELECT relname, indexrelname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0;
-- MySQL
SELECT * FROM sys.schema_unused_indexes;
```

```js
db.coll.aggregate([{ $indexStats: {} }])
```

`idx_scan = 0` is only meaningful if statistics have accumulated since well before the last reset and
cover a full business cycle, including monthly and quarterly jobs. **Never drop an unused unique or
FK-supporting index on that evidence alone.**

---

## SQLite uses one index per table per query

**Signal:** A query with two independently indexed predicates ANDed together. An index on a `TEXT`
column searched with `LIKE 'abc%'`. A schema with no `ANALYZE` anywhere in the migration or startup
path.

**Why it breaks:** SQLite uses at most one index per table per query for AND terms — it explicitly does
not implement AND-by-intersect — so two single-column indexes never combine. You need one composite
index. And without `ANALYZE`, `sqlite_stat1` is empty and the choice between candidate indexes is
essentially arbitrary, which is how a query silently picks the useless one.

**Fix:**

```sql
CREATE INDEX idx_f_s ON FruitsForSale(fruit, state, price);   -- one composite
ANALYZE;                       -- after bulk load or in deploy, not per request
```

The `LIKE` prefix optimisation needs the default `case_sensitive_like=OFF` and the column declared
`COLLATE NOCASE` — or use `GLOB`.

**Fine when:** the terms are OR-connected. SQLite can union results from several indexes there, so
multiple single-column indexes are correct for `WHERE a = ? OR b = ?`.

`PRAGMA optimize` on connection close is the modern, cheap alternative to a full `ANALYZE`.

---

## Sources

PostgreSQL: [indexes-index-only-scans](https://www.postgresql.org/docs/current/indexes-index-only-scans.html),
[indexes-types](https://www.postgresql.org/docs/current/indexes-types.html),
[indexes-multicolumn](https://www.postgresql.org/docs/current/indexes-multicolumn.html),
[indexes-partial](https://www.postgresql.org/docs/current/indexes-partial.html),
[sql-createindex](https://www.postgresql.org/docs/current/sql-createindex.html).
MySQL: [create-index](https://dev.mysql.com/doc/refman/8.4/en/create-index.html),
[index-extensions](https://dev.mysql.com/doc/refman/8.4/en/index-extensions.html),
[foreign-keys](https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html).
MongoDB: [equality-sort-range](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/),
[index-partial](https://www.mongodb.com/docs/manual/core/index-partial/),
[index-sparse](https://www.mongodb.com/docs/manual/core/index-sparse/).
