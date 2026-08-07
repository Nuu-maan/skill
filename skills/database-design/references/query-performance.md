# Query performance

Query shapes that break at scale, detectable by reading the code. Load when reviewing query code, ORM
calls, pagination, or anything the user reports as slow.

For interpreting a query plan the user pastes in, use [reading-plans.md](reading-plans.md) instead.

Every entry has a **Fine when** line. Read it before reporting.

## Contents

- `NOT IN` with a nullable subquery
- `SELECT *`
- `OFFSET` pagination
- `count(*)` on a large table
- Type mismatch between parameter and column
- `LIKE '%term%'` and unanchored search
- `DISTINCT` hiding a join fan-out
- Queries with no `LIMIT`
- `ORDER BY` on an unindexed column
- Huge `IN` lists
- Correlated subqueries in the select list
- MongoDB `$where` and unanchored `$regex`
- MongoDB pipeline stage order

---

## `NOT IN` with a nullable subquery

**Signal:** `WHERE x NOT IN (SELECT y FROM t)` where `t.y` is nullable, or `NOT IN` over an array built
in application code that may contain `null`.

**Why it breaks:** If the subquery returns even one NULL, `x NOT IN (...)` evaluates to NULL — never
TRUE — for **every** row, so the query returns zero rows. No error, no warning. This is the classic
"my filter silently returns nothing in production" bug, and it appears only after the first NULL lands
in that column. `IN` is unaffected; only the negation flips.

**Fix:**

```sql
SELECT * FROM users u
WHERE NOT EXISTS (SELECT 1 FROM bans b WHERE b.user_id = u.id);

-- or, keeping NOT IN
... NOT IN (SELECT y FROM t WHERE y IS NOT NULL)
```

**Fine when:** the inner column is declared `NOT NULL` — check the DDL, not the ORM model, since an
ORM `required` flag does not always emit a database constraint — or it is the primary key. Also fine
for a hardcoded literal list with no NULLs.

`NOT EXISTS` also plans better: it gives an anti-join, which `NOT IN` over a nullable column often
cannot.

**Confirm it:** `SELECT count(*) FROM t WHERE y IS NULL;`

---

## `SELECT *`

**Signal:** `SELECT *` in raw SQL, or an ORM call with no field list — Prisma `findMany()` with no
`select`, Django `Model.objects.all()` with no `.only()`, Sequelize `findAll()` with no `attributes`,
`db.coll.find(filter)` with no projection.

**Why it breaks:** Every column is read and shipped, including TOASTed and BLOB columns stored
off-page, so a row with one large `text`/`jsonb`/`bytea` column costs an extra fetch per row. It also
destroys index-only scans — Postgres and MySQL can answer from a covering index only if the select
list is a subset of the index columns. In an API handler it silently leaks columns added by later
migrations: password hashes, internal flags.

**Fix:**

```sql
SELECT id, email, created_at FROM users WHERE org_id = $1;
```

```ts
prisma.user.findMany({ select: { id: true, email: true } })
db.users.find({ orgId }, { _id: 1, email: 1 })
```

```python
User.objects.only("id", "email")        # or .values("id", "email")
```

**Fine when:** the row is narrow (all fixed-width scalars, no text/json/blob), or you genuinely need
every column — a full-record edit form, an ETL export.

Do **not** flag `SELECT *` inside `EXISTS (...)` — the select list there is ignored by every engine.
Do not flag `SELECT count(*)`.

---

## `OFFSET` pagination

**Signal:** `LIMIT n OFFSET n*page`, Prisma `{ skip, take }`, Django `Paginator` or `qs[100:120]`,
Mongo `.skip(n).limit(m)` — where the offset comes from a user-supplied page number.

**Why it breaks:** PostgreSQL's own documentation: "The rows skipped by an OFFSET clause still have to
be computed inside the server; therefore a large OFFSET might be inefficient." Cost grows linearly with
page depth, so page 5000 reads and discards ~100k rows on every request — an easy accidental DoS on a
public endpoint. Rows shifting between requests also duplicate or skip items, because OFFSET is not a
stable position.

**Fix:** keyset (cursor) pagination, with an index on `(created_at DESC, id DESC)`.

```sql
SELECT id, created_at FROM posts
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

```ts
findMany({ take: 20, skip: 1, cursor: { id: lastId }, orderBy: { id: 'desc' } })
```

**Fine when:** the offset is bounded and small — an admin table capped at a few dozen pages, a UI that
only shows pages 1–10 — or the table is under ~10k rows, where the scan is cheap and keyset adds real
complexity.

Keyset pagination **cannot support "jump to page 47."** If the product requires numbered page jumps,
OFFSET is the correct trade-off. Say so rather than recommending a rewrite that breaks the feature.

---

## `count(*)` on a large table

**Signal:** `SELECT count(*) FROM big_table` with no `WHERE`, usually for a `totalPages` field. ORM
forms: Prisma `count()`, Django `.count()` (and `Paginator`, which calls it on every page), Mongo
`countDocuments({})`.

**Why it breaks:** PostgreSQL MVCC has no stored row count — `count(*)` must visit every row or index
entry, and an index-only scan still needs visibility-map checks. InnoDB is the same; its
`SHOW TABLE STATUS` row count is an estimate. MongoDB's docs state `countDocuments()` with an empty
filter "performs a full collection scan." On 10M rows this is seconds per request, running alongside
the page query.

**Fix:**

```sql
SELECT reltuples::bigint FROM pg_class WHERE relname = 'events';   -- estimate, sub-ms
SELECT count(*) FROM (SELECT 1 FROM events LIMIT 1000) t;          -- or bound it
```

```js
db.events.estimatedDocumentCount()
```

```python
qs.exists()          # not  if qs.count() > 0
```

**Fine when:** the table is known-small, the count is heavily filtered on a selective indexed
predicate, or the value is cached or materialised.

`count(*)` is **not** slower than `count(1)` or `count(id)` in PostgreSQL or MySQL. Do not report that
— it is folklore.

---

## Type mismatch between parameter and column

**Signal:** `WHERE user_id = '42'` against an integer column, or `WHERE order_no = 42` against a
`varchar` column. In practice this comes from `req.params.id` (always a string) passed straight
through.

**Why it breaks:** MySQL's docs: "For comparisons of a string column with a number, MySQL cannot use an
index on the column to look up the value quickly" — both sides are cast to double and the index is
dead. Worse, `WHERE varchar_col = 0` matches every row that converts to 0 (`'apple'` → 0), returning
the whole table even in strict mode.

**Fix:**

```sql
SELECT * FROM t WHERE order_no = '0';    -- quote to match the column's type
```

```ts
const id = Number(req.params.id);
if (!Number.isInteger(id)) return 400;   // coerce at the edge, not in SQL
```

**Fine when:** on PostgreSQL, which does **not** do this silently — comparing `text = integer` raises
`operator does not exist`, so it fails loudly in CI rather than lurking. SQLite applies column affinity,
so an int column compared to `'42'` usually still uses the index. The reverse direction in MySQL (int
column vs `'42'`) is also safe — MySQL casts the literal and keeps the index.

**Only the MySQL string-column-vs-number direction is serious.** The PostgreSQL analogue to watch for
is a function-wrapped column (`WHERE lower(email) = $1`), which needs a matching expression index — see
[indexes.md](indexes.md).

---

## `LIKE '%term%'` and unanchored search

**Signal:** `LIKE '%term%'`, `LIKE CONCAT('%', ?, '%')`, Prisma `{ contains: q }`, Django
`__icontains`, Sequelize `Op.substring`. Also `LIKE 'foo%'` on a PostgreSQL database not in the C
locale.

**Why it breaks:** Postgres uses a B-tree index for `LIKE` only "if the pattern is a constant and is
anchored to the beginning of the string" — a leading `%` forces a full scan. Separately, even an
anchored `LIKE 'foo%'` will **not** use a plain B-tree index unless the database is in the C locale or
the index was built with `text_pattern_ops`. A search box that felt instant on 1k rows becomes a
sequential scan at 1M.

**Fix:**

```sql
CREATE INDEX ON users (email text_pattern_ops);      -- anchored prefix, non-C collation

CREATE EXTENSION pg_trgm;                            -- true substring search
CREATE INDEX ON products USING gin (name gin_trgm_ops);
```

MySQL 8: `FULLTEXT` plus `MATCH ... AGAINST` for word search.

**Fine when:** the table is a small lookup (country list, tag vocabulary), or another selective indexed
predicate already reduces the candidate set to a few hundred rows.

SQLite has its own trap: the LIKE optimisation applies only if `case_sensitive_like` is off **and** the
column uses `NOCASE` collation, the pattern has no leading wildcard, and the column has TEXT affinity.
An anchored LIKE on a default-collation SQLite column is often still a scan.

---

## `DISTINCT` hiding a join fan-out

**Signal:** `SELECT DISTINCT` (or Prisma `distinct:`, Django `.distinct()`) on a query that also joins
a one-to-many relation.

**Why it breaks:** The `DISTINCT` is usually not a requirement — it is a bandage over a join fan-out.
The join multiplies rows, one per child, the engine sorts or hashes the whole inflated set to dedupe,
and **any aggregate in the query is now silently wrong**: `SUM(orders.total)` counts each order once
per matching item row.

**Fix:**

```sql
-- replace the fan-out join with a semi-join
SELECT u.id, u.email FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'paid');

-- if you need the aggregate, aggregate before joining
JOIN (SELECT user_id, sum(total) t FROM orders GROUP BY 1) s ON s.user_id = u.id
```

**Fine when:** genuinely deduplicating values from a single table (`SELECT DISTINCT status FROM
orders`), or on a `UNION` of independent sources. Postgres `DISTINCT ON (...)` is a deliberate
top-1-per-group idiom, not a bandage.

**Highest-signal combination:** `DISTINCT` + `JOIN` + an aggregate in the same statement. That is a
probable correctness bug, not just slowness — report it as such.

---

## Queries with no `LIMIT`

**Signal:** A query with a `WHERE` on user input and no `LIMIT`/`take`/`.limit()`, inside a request
handler. Also `findAll()`, `objects.filter(...)` returned straight to a serializer,
`db.coll.find({...}).toArray()`.

**Why it breaks:** Result size is whatever the data happens to be. Fine at 200 rows in development; a
memory spike and a multi-second response once one tenant has 500k. It is also the standard shape of an
accidental full-table dump through a public endpoint, because a missing or empty filter parameter often
widens to "everything."

**Fix:**

```ts
const rows = await prisma.event.findMany({
  where, orderBy: { id: 'desc' },
  take: Math.min(limit ?? 50, 200),      // always cap
});
```

**Fine when:** the query is by primary key or a unique constraint (at most one row), it is a background
job that intentionally processes everything (ideally batched), or the `WHERE` clause is provably
bounded. Do not flag aggregate queries that return one row.

---

## `ORDER BY` on an unindexed column

**Signal:** `ORDER BY col` where `col` appears in no index, especially with `LIMIT` for a "latest N"
list. Also `ORDER BY RANDOM()` / `ORDER BY RAND()`.

**Why it breaks:** Without a matching index the engine reads the full candidate set and sorts it. In
Postgres, when the sort exceeds `work_mem` (default 4MB) it spills to disk as an external merge sort —
orders of magnitude slower, and invisible until the data grows. `ORDER BY RANDOM()` is worse: it
assigns a random value to every row and sorts all of them, so even `LIMIT 1` scans the whole table.

**Fix:**

```sql
CREATE INDEX ON posts (author_id, created_at DESC);
SELECT ... WHERE author_id = $1 ORDER BY created_at DESC LIMIT 20;

-- a cheap random row
SELECT * FROM posts
WHERE id >= (random() * (SELECT max(id) FROM posts))::bigint
ORDER BY id LIMIT 1;
```

**Fine when:** a selective indexed `WHERE` already reduced the set — sorting 50 rows in memory is free
— or the table is a small reference table.

The index must match **direction and column order**. An index on `(created_at)` does not serve
`ORDER BY author_id, created_at DESC`, and a mixed `ORDER BY a ASC, b DESC` needs an index declared with
those same directions.

---

## Huge `IN` lists

**Signal:** A parameter list built by joining an array — `WHERE id IN (${ids.join(',')})`, Prisma
`{ id: { in: ids } }`, Drizzle `inArray(t.id, ids)`, Mongo `{ _id: { $in: ids } }` — where `ids` comes
from a previous query's results.

**Why it breaks:** This is usually the visible half of an N+1 that was half-fixed. PostgreSQL's wire
protocol encodes the parameter count as an unsigned Int16, so the query hits a hard
`number of parameters must be between 0 and 65535` error — a production 500 that never appears in
development. MySQL abandons range optimisation and falls back to a full table scan once the IN list
exceeds `range_optimizer_max_mem_size` (default 8 MB), emitting only warning 3170.

**Fix:**

```sql
-- one round trip, one parameter, no limit
SELECT * FROM items WHERE id = ANY($1::bigint[]);

-- or join against the source query instead of materialising ids
SELECT i.* FROM items i JOIN carts c ON c.item_id = i.id WHERE c.user_id = $1;
```

**Fine when:** the list is small and bounded — a status enum, a handful of ids from a UI multi-select.
A short IN list is well optimised everywhere.

PostgreSQL 18 added transforming OR-clauses to arrays for faster index processing, so long
`a = 1 OR a = 2 OR ...` chains plan like `= ANY` from 18 onward. On 17 and earlier, rewrite them
yourself.

---

## Correlated subqueries in the select list

**Signal:** A subquery in the `SELECT` list referencing the outer row:
`SELECT u.id, (SELECT count(*) FROM orders o WHERE o.user_id = u.id) AS n FROM users u`. The ORM form
is computing a count inside a `.map()` over already-fetched rows.

**Why it breaks:** The subquery is evaluated once per output row. At 10 users it is invisible; at 100k
users it is 100k index lookups inside one statement, and `EXPLAIN` shows it as a single
innocuous-looking node with a high `loops` count. **This is an N+1 that lives inside the database**, so
ORM-level N+1 detectors never see it.

**Fix:**

```sql
SELECT u.id, COALESCE(o.n, 0) AS n
FROM users u
LEFT JOIN (SELECT user_id, count(*) AS n FROM orders GROUP BY user_id) o
  ON o.user_id = u.id;

-- or a window function when you need per-row context
SELECT id, count(*) OVER (PARTITION BY user_id) FROM orders;
```

**Fine when:** the outer query returns a handful of rows — a record detail page, anything with a small
`LIMIT` applied first — or the correlated subquery is in the `WHERE` clause as `EXISTS`, which is a
semi-join and planned efficiently rather than row by row.

Postgres can sometimes flatten simple correlated subqueries; MySQL is far less likely to.

---

## MongoDB `$where` and unanchored `$regex`

**Signal:** `$where: function() {...}`. `$regex` without a leading `^`, or built from user input via
`new RegExp(q)`. `$regex` with the `i` option on a field with a case-insensitive index.

**Why it breaks:** MongoDB's docs state `$where` "evaluates JavaScript and cannot take advantage of
indexes" and requires a collection scan; it is deprecated as of MongoDB 8.0, which logs a warning on
every use. An unanchored `$regex` likewise scans, since only a prefix expression can become an index
range. And case-insensitive indexes "do not improve performance for `$regex` queries, as the `$regex`
operator is not collation-aware."

**Fix:**

```js
db.users.find({ email: { $regex: /^acme/ } })                    // prefix, uses the index

db.orders.find({ $expr: { $gt: ["$total", "$limit"] } })         // replaces $where, no JS

db.products.createIndex({ name: "text" })                        // real word search
db.products.find({ $text: { $search: "widget" } })
```

**Fine when:** the collection is small, it is an admin-only script, or another indexed equality
predicate in the same filter (`{ tenantId, name: /foo/ }`) narrows the candidates first — MongoDB
applies the regex to the survivors. An anchored, case-sensitive `/^prefix/` on an indexed field is
correct and needs no change.

A `RegExp` built by interpolating a request value is also a ReDoS vector — flag it for
`backend-hardening` as well.

---

## MongoDB pipeline stage order

**Signal:** An aggregation pipeline whose first stage is `$project`, `$unwind`, `$addFields`,
`$lookup`, or `$group`, with `$match` appearing later. Or a `$sort` placed after `$project`, `$unwind`,
or `$group`.

**Why it breaks:** Per MongoDB's docs, "the server can use an index if `$match` is the first stage in
the pipeline," and `$sort` can use an index only if "the stage is not preceded by a `$project`,
`$unwind`, or `$group` stage." Put a `$project` first and the whole collection is read and reshaped in
memory before anything is filtered; the in-memory sort then hits the 100 MB per-stage limit and fails.

**Fix:**

```js
db.orders.aggregate([
  { $match:  { tenantId, status: "paid" } },   // first, uses the index
  { $sort:   { createdAt: -1 } },              // still indexable
  { $limit:  20 },                             // coalesced into $sort
  { $lookup: { /* ... */ } },                  // after narrowing
  { $project: { /* ... */ } }
])
```

**Fine when:** the earlier stage is required for correctness — `$unwind` before matching on a computed
element, or `$geoNear`, which must be first and always uses its geospatial index. Also fine on a small
collection or a deliberate full-collection reporting job.

`$sort` followed by `$limit` is coalesced by the optimiser **only** if no intervening stage changes the
document count. Inserting a `$lookup` between them silently removes that optimisation.

---

## Sources

PostgreSQL: [queries-limit](https://www.postgresql.org/docs/current/queries-limit.html),
[indexes-types](https://www.postgresql.org/docs/current/indexes-types.html),
[protocol-message-formats](https://www.postgresql.org/docs/current/protocol-message-formats.html),
[release-18](https://www.postgresql.org/docs/18/release-18.html).
MySQL: [type-conversion](https://dev.mysql.com/doc/refman/8.4/en/type-conversion.html),
[range-optimization](https://dev.mysql.com/doc/refman/8.4/en/range-optimization.html).
SQLite: [optoverview](https://www.sqlite.org/optoverview.html).
MongoDB: [$where](https://www.mongodb.com/docs/manual/reference/operator/query/where/),
[$regex](https://www.mongodb.com/docs/manual/reference/operator/query/regex/),
[countDocuments](https://www.mongodb.com/docs/manual/reference/method/db.collection.countDocuments/),
[aggregation-pipeline-optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/).
