# Reading query plans

How to interpret a plan the user pastes in. Load only when you have actual plan output — this skill
never runs a query itself.

**Ask for the right command first.** A plan of estimates cannot support a performance claim.

| Engine | Ask the user to run |
| :--- | :--- |
| PostgreSQL | `EXPLAIN (ANALYZE, BUFFERS) SELECT ...;` (BUFFERS is automatic from PG 18) |
| MySQL 8.0.18+ | `EXPLAIN ANALYZE SELECT ...;` |
| MySQL older | `EXPLAIN FORMAT=JSON SELECT ...;` — estimates only, say so |
| SQLite | `EXPLAIN QUERY PLAN SELECT ...;` and `SELECT * FROM sqlite_stat1;` |
| MongoDB | `db.coll.find(...).explain("executionStats")` — the argument is required |

**Every entry below has a numeric threshold.** Do not report a plan observation as a finding until it
crosses the threshold. Plans are full of scary-looking nodes that are working correctly.

## Contents

**PostgreSQL** — Seq Scan with high Rows Removed · Estimate vs actual · Lossy bitmap heap scan ·
Nested Loop with high loops · Sort spilling to disk · Buffers read vs hit
**MySQL** — the `type` column · `Using temporary; Using filesort` · `Using index` vs `Using where` ·
Plain EXPLAIN is estimates only
**SQLite** — `SCAN` vs `SEARCH` · `USE TEMP B-TREE` and `AUTOMATIC INDEX`
**MongoDB** — COLLSCAN and the examined:returned ratio · blocking `SORT` stage

---

## PostgreSQL

### Seq Scan with high Rows Removed by Filter

**Signal:**

```
Seq Scan on orders  (actual rows=12.00 loops=1)
  Filter: (user_id = 42)
  Rows Removed by Filter: 998812
```

**What it means:** the engine read every row and threw almost all away. This is the single most common
missing-index signature in Postgres plans.

**Threshold:** report when `Rows Removed by Filter ÷ (actual rows + 1) > 100`, **or** `Rows Removed by
Filter > 50,000`. Below that, report nothing.

**Fix:** index exactly the columns on the `Filter:` line, equality first.

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_created ON orders (user_id, created_at DESC);
```

**Not a finding when:** the table is under ~1,000 rows, or the query genuinely returns a large fraction
of the table. If the filter is unindexable as written (`lower(email) = $1`), the fix is an expression
index, not a plain one.

---

### Estimate vs actual divergence

**Signal:** `(cost=... rows=50 ...) (actual time=... rows=48000.00 loops=1)` — compare `rows=` inside
the cost parens against `rows=` inside the actual parens.

**What it means:** the planner chose join order, join method, and scan type from the estimate. A
1000× low estimate gets you a Nested Loop where a Hash Join was needed. **The plan is not the bug —
the statistics that produced it are.**

**Threshold:** ratio ≥ 10× in either direction **and** actual rows ≥ 1000. Small-number ratios are
noise.

**Fix:**

```sql
ANALYZE orders;

ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;   -- skewed column
ANALYZE orders;

CREATE STATISTICS orders_geo (dependencies, ndistinct) ON city, country FROM orders;
ANALYZE orders;                                               -- correlated columns
```

**Three mismatches are expected and must NOT be reported:**

1. A node under `LIMIT` stops early, so actual < estimate by design.
2. **Any node with `loops` > 1 reports actual rows as a per-loop average.** Multiply before comparing.
3. The inner side of a Merge Join re-emits rows on duplicate keys, counted as real rows.

**Confirm staleness:**

```sql
SELECT relname, n_live_tup, n_mod_since_analyze, last_analyze, last_autoanalyze
  FROM pg_stat_user_tables ORDER BY n_mod_since_analyze DESC LIMIT 20;
```

Autoanalyze only fires after 50 rows + 10% of the table changes, so a 100M-row table needs 10M changes
first and is routinely stale.

---

### Lossy bitmap heap scan

**Signal:** `Heap Blocks: lossy=90` — any non-zero `lossy=`, often with `Rows Removed by Index
Recheck`.

**What it means:** the bitmap did not fit in `work_mem`, so Postgres degraded from tracking individual
tuples to tracking whole 8KB pages. Every row on those pages is read and re-tested — a partial
sequential scan hiding inside what looks like an index plan.

**Threshold:** `lossy=` present and non-zero. **`exact=N` alone is healthy — ignore it.**
`Recheck Cond:` on its own is normal on every Bitmap Heap Scan and is not a problem.

**Fix:** make the index selective enough that the bitmap stays small, or raise `work_mem` for the
session.

```sql
SET work_mem = '64MB';
CREATE INDEX CONCURRENTLY idx_events_tenant_created ON events (tenant_id, created_at);
```

**Never recommend a global `work_mem` increase** — it is allocated per sort/hash node per connection
and multiplies.

---

### Nested Loop with high `loops`

**Signal:** `-> Index Scan using idx_b on b (actual time=0.03..0.03 rows=1.00 loops=250000)`

**What it means:** per-loop time looks trivial, so this node reads as fast. Total time is
0.03 × 250,000 = **7.5 seconds**. Postgres reports averages deliberately, and readers who do not
multiply misdiagnose the plan.

**Threshold:** compute `actual time (second number) × loops` for every node with `loops` > 1 and rank
the plan by that, not by printed time. Flag when one node's computed total exceeds 25% of the top-level
`Execution Time`.

**Fix:** fix the estimate that caused the Nested Loop choice — `ANALYZE` the outer table, or make the
join key indexable.

`SET enable_nestloop = off` is a **diagnostic switch to confirm the theory only.** Never ship it in
application code or `postgresql.conf`; it will make other queries worse.

**Not a finding when:** the inner side is a true single-row primary-key lookup and the outer side is
small. The totals stay tiny.

---

### Sort spilling to disk

**Signal:** `Sort Method: external merge  Disk: 214880kB` rather than
`Sort Method: quicksort  Memory: 3096kB`. Also `HashAggregate` reporting `Disk Usage:`.

**What it means:** the sort exceeded `work_mem` and spilled to temporary files — one to two orders of
magnitude slower. `work_mem` defaults to 4MB, which almost nothing in a real application fits into.

**Threshold:** report when `Disk` is present at all. Escalate when the spill exceeds ~50MB or the sort
node is inside a loop.

**Fix:** remove the sort with an index that supplies the order. That is better than raising memory.

```sql
CREATE INDEX CONCURRENTLY idx_posts_feed ON posts (author_id, created_at DESC);
```

If memory is genuinely needed, scope it: `ALTER ROLE reporting SET work_mem = '256MB'` — not globally.
A one-off analytical report that sorts millions of rows will always spill, and that is acceptable.

---

### Buffers: read far exceeding hit

**Signal:** `Buffers: shared hit=12 read=184320` on the expensive node.

**What it means:** `hit` is pages found in shared buffers; `read` is pages fetched from the OS or disk.
Blocks are 8KB, so `read=184320` is roughly 1.4GB of I/O for one query. This is the metric that
explains why a query is fast on a warm laptop and slow in production.

**Threshold:** one node's `read` exceeds 100,000 blocks (~800MB), or `read` is more than 10× `hit` on
the dominant node.

**Not a finding when:** it is the first execution on a cold cache — high `read`, low `hit` is expected.
**Ask for a second run before concluding.** `read` also counts OS page-cache hits, so real disk I/O may
be much lower than it appears.

---

## MySQL

### The `type` column

Worst to best: `ALL` < `index` < `range` < `ref` < `eq_ref` < `const` < `system`. `index_merge`,
`ref_or_null`, and `fulltext` sit between `range` and `ref`.

**The trap is `index`.** It looks like an index is being used, but it is a full scan *of the index* and
only marginally better than `ALL`.

**Threshold:** report `ALL` when that row's `rows` estimate exceeds ~1,000, or when it appears on any
table that is not first in the join order. Report `index` **unless** `Extra` also says `Using index`.

**Fix:** read `possible_keys` and `key` — `NULL` means no usable index exists.

```sql
ALTER TABLE orders ADD INDEX idx_user_status (user_id, status);
ALTER TABLE users ADD UNIQUE KEY uq_users_email (email);   -- eq_ref/const need PRIMARY or UNIQUE NOT NULL
```

**Not a finding when:** `type: ALL` on a small lookup or config table — correct and cheaper than an
index. `type: index` with `Extra: Using index` is a covering index scan and is often the intended plan.

`rows` is an InnoDB estimate from sampled index dives and can be off by an order of magnitude. Never
treat it as a count.

---

### `Using temporary; Using filesort`

**Signal:** both strings on the same `Extra` row.

**What it means:** MySQL materialised an internal temporary table then sorted it — typically because
`GROUP BY` and `ORDER BY` name different columns, or `ORDER BY` spans columns from different tables. If
the temp table exceeds `tmp_table_size` it converts to an on-disk InnoDB table and latency jumps.

**Threshold:** both on one row, or `Using filesort` on a row whose `rows` exceeds ~10,000.

**`Using filesort` does not mean files or disk.** It is MySQL's name for any sort not satisfied by an
index, done in memory for small result sets. On a query sorting a few dozen rows it is irrelevant.
`Using temporary` is also unavoidable for `UNION` and some `DISTINCT` forms.

**Fix:** align `GROUP BY` and `ORDER BY`, then index that order. Or add `ORDER BY NULL` when order is
genuinely not needed.

**Confirm the disk spill:** `SHOW SESSION STATUS LIKE 'Created_tmp_disk_tables';` before and after.

---

### `Using index` vs `Using index condition` vs `Using where`

These three look interchangeable and mean very different things:

| `Extra` | Meaning |
| :--- | :--- |
| `Using index` | Covering index, no table lookup. **Good — never flag this.** |
| `Using index condition` | Index Condition Pushdown: index used, part of the WHERE pushed down, rows still fetched. |
| `Using where` + `type: ALL` + `key: NULL` | Filtering happens after a full scan. **This is the finding.** |

`Using where` on its own is normal on almost every filtered query and is not a finding.

A covering index is not free — it widens the index, slows writes, and duplicates column data. Only
recommend one for a demonstrably hot query. MySQL's covering-index optimisation does not apply to
`TEXT`/`BLOB` columns.

---

### Plain EXPLAIN is estimates only

If the pasted output has columns `id | select_type | table | type | ...`, it is estimate-only. `rows`
comes from sampled statistics and `filtered` is a guessed percentage, so the product across the join is
a rough upper bound, not a measurement.

**Ask for `EXPLAIN ANALYZE` (MySQL 8.0.18+) before making a performance claim.** Meanwhile, multiply
the `rows` column across all rows; treat a product above ~100,000 as worth investigating.

```sql
EXPLAIN ANALYZE SELECT ...;
ANALYZE TABLE orders;                        -- if the estimates look absurd
EXPLAIN FORMAT=JSON FOR CONNECTION 1234;     -- a query already running elsewhere
```

**`EXPLAIN ANALYZE` actually runs the statement.** Never suggest it for `UPDATE`/`DELETE` on production
without a transaction the user will roll back. It supports only `SELECT`, `TABLE`, and multi-table
`UPDATE`/`DELETE`, cannot combine with `FOR CONNECTION`, and errors on `FORMAT=JSON`.

---

## SQLite

### `SCAN` vs `SEARCH`

```
SCAN orders                                        -- bad: every row
SEARCH orders USING INDEX idx_user (user_id=?)     -- good: a subset
SEARCH orders USING INTEGER PRIMARY KEY (rowid=?)  -- best
```

SQLite gives no cost numbers at all, so this one word is the entire diagnosis. In a join, an inner-loop
`SCAN` runs once per outer row — an N-row scan inside an M-row loop is N × M row visits.

**Threshold:** report a `SCAN` on any table the app writes rows to over time, or any `SCAN` that is not
the outermost loop.

**Fix:**

```sql
CREATE INDEX idx_orders_user ON orders(user_id);
ANALYZE;     -- without sqlite_stat1 the planner guesses and may keep choosing SCAN
```

**Check whether stats exist at all:** `SELECT * FROM sqlite_stat1;` — empty or missing means `ANALYZE`
has never run.

**Not a finding when:** `SCAN` on the outermost table of a query with no `WHERE`, on a small static
lookup table, or when the query genuinely returns most rows. Indexing those makes things slower.

> The output format is explicitly documented as unstable. SQLite 3.36.0 dropped the leading table
> numbers and the `TABLE`/`SUBQUERY` keywords, so older tutorials show `SCAN TABLE t1` where current
> versions print `SCAN t1`. Do not rely on the older shape.

---

### `USE TEMP B-TREE` and `AUTOMATIC INDEX`

**`USE TEMP B-TREE FOR ORDER BY`** — SQLite built a throwaway sort structure because no index supplied
the order.

**`USING AUTOMATIC INDEX`** — SQLite decided building a whole transient index at query time was cheaper
than scanning. **This is SQLite telling you a permanent index is missing**, and it pays that build cost
on every execution.

**Threshold:** any `AUTOMATIC INDEX` line is a finding. `USE TEMP B-TREE` is a finding only when the
query has no `LIMIT` or the table is large — over a few hundred rows it is often optimal, and building
an index to avoid it is a net loss.

**`USING COVERING INDEX` is the good case and must never be flagged** — every column the query needs is
in the index, so SQLite skips the table B-tree entirely.

**Fix:**

```sql
CREATE INDEX idx_posts_feed ON posts(author_id, created_at DESC);   -- match order and direction
CREATE INDEX idx_items_order ON items(order_id);                    -- the column AUTOMATIC INDEX named
```

`PRAGMA automatic_index=OFF` removes the warning, not the missing index.

---

## MongoDB

### COLLSCAN and the examined-to-returned ratio

**Signal:** `winningPlan` contains `"stage": "COLLSCAN"`, or the counters show a large gap:
`totalDocsExamined: 2400000, nReturned: 20`.

**What it means:** the ratio is the single headline number. Ideal is 1:1. MongoDB Atlas raises its
"Query Targeting: Scanned Objects / Returned" alert at **1000:1** by default.

**Threshold:** compute `totalDocsExamined ÷ nReturned` and `totalKeysExamined ÷ nReturned`. Report when
either exceeds 100 and `nReturned` ≥ 1. Escalate to critical above 1000.

`totalKeysExamined` much larger than `nReturned` **with an IXSCAN present** means the index exists but
its field order is wrong for this query — see the ESR rule in [indexes.md](indexes.md).

**Fix:**

```js
db.orders.createIndex({ userId: 1, createdAt: -1 });   // Equality, Sort, Range
```

**Not a finding when:** an unfiltered `find({})`, an aggregation whose first stage is a full pass, a
collection under a few thousand documents, or a field with no selective values. A ratio of 1 with
`nReturned: 0` is meaningless — ignore empty result sets.

> `explain()` **without** the `"executionStats"` argument defaults to `queryPlanner` verbosity and
> returns no `nReturned` or `totalDocsExamined` at all. If those fields are absent, ask for a re-run
> rather than guessing.

---

### Blocking `SORT` stage

**Signal:** a `SORT` stage in `executionStages` (often above `SORT_KEY_GENERATOR`), or the error
`Sort exceeded memory limit of 104857600 bytes, but did not opt in to external sorting`.

**What it means:** MongoDB fetched every matching document before it could return the first one. The
sort is capped at 100MB and **fails hard rather than degrading** — so it works in development and
errors in production.

**Threshold:** a `SORT` stage present **and** its input `nReturned` exceeds ~1,000, or `usedDisk` /
`spills` / `spilledBytes` is non-zero (reported from MongoDB 6.0).

**Fix:**

```js
db.posts.createIndex({ authorId: 1, createdAt: -1 });   // removes the SORT stage
```

Sort elimination requires the sort direction to **match the index or be its exact inverse across all
keys**. A mixed `{a: 1, b: -1}` sort will not use an `{a: 1, b: 1}` index — always state the index
direction explicitly.

`allowDiskUse: true` removes the error but keeps the slowness. Prefer the index.

**Not a finding when:** the sort is over a small already-limited set — a `$limit` of 20 pushed below
it, or `PROJECTION_COVERED` feeding it.

---

## Sources

PostgreSQL: [using-explain](https://www.postgresql.org/docs/current/using-explain.html),
[runtime-config-autovacuum](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html).
MySQL: [explain-output](https://dev.mysql.com/doc/refman/8.4/en/explain-output.html),
[explain](https://dev.mysql.com/doc/refman/8.4/en/explain.html).
SQLite: [eqp](https://www.sqlite.org/eqp.html).
MongoDB: [explain-results](https://www.mongodb.com/docs/manual/reference/explain-results/),
[query-targeting](https://www.mongodb.com/docs/atlas/reference/alert-resolutions/query-targeting/).
