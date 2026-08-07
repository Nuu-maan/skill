# SQLite

Load when the project uses SQLite, Turso/libSQL, Cloudflare D1, or `better-sqlite3`. One engine file per
review.

SQLite is the only engine of the four where **the schema file can be entirely correct and enforce
nothing.** Foreign keys are off by default, declared types are advisory, and the primary key accepts
NULL. Check the connection setup before you review a single column.

## Contents

- The five most common defects
- Foreign keys are off by default
- Type affinity
- `INT PRIMARY KEY` is not `INTEGER PRIMARY KEY`
- Indexes
- Migrations: the 12-step rebuild
- Reading `EXPLAIN QUERY PLAN`
- Concurrency and deployment
- Grep list

---

## The five most common defects in a SQLite schema

1. **`PRAGMA foreign_keys = ON` is never issued**, so every declared foreign key is decorative.
2. **Declared types SQLite does not have** — `DECIMAL(10,2)`, `BOOLEAN`, `DATETIME` — silently mapped by
   affinity, with money landing in a float.
3. **`id INT PRIMARY KEY`** instead of `INTEGER PRIMARY KEY`, so there is no rowid alias and no
   auto-assignment.
4. **No `ANALYZE` anywhere**, so `sqlite_stat1` is empty and index choice is close to arbitrary.
5. **Two single-column indexes for an ANDed query** — SQLite uses at most one index per table per query.

---

## Foreign keys are off by default

**This is the finding that matters most.** SQLite parses and stores foreign key definitions but does not
enforce them unless the pragma is set — and **the setting is per connection**, not per database. Every
pooled connection, every `sqlite3` CLI session, and every new worker starts at 0.

```sql
PRAGMA foreign_keys = ON;      -- immediately after opening EVERY connection
PRAGMA foreign_key_check;      -- audit what already leaked through
```

```js
db.pragma('foreign_keys = ON')                    // better-sqlite3
```

```python
@event.listens_for(Engine, 'connect')             # SQLAlchemy
def _fk_on(conn, _): conn.execute('PRAGMA foreign_keys=ON')
```

**Fine when the driver does it for you:** Django's SQLite backend, the Rails sqlite3 adapter,
`bun:sqlite`, and Prisma all set it per connection. Check for one of those before flagging.

**Also flag `PRAGMA foreign_keys` issued inside an open transaction** — it is a silent no-op there.

---

## Type affinity

SQLite ignores the parenthesised size and maps the declared type to one of five affinities by substring
match. `DECIMAL(10,2)`, `BOOLEAN`, and `DATETIME` all get NUMERIC affinity — **so a money value is stored
as a float** — and `VARCHAR(50)` accepts a 10,000-character string without error.

```sql
CREATE TABLE items (
  id          INTEGER PRIMARY KEY,        -- exactly INTEGER, not INT
  name        TEXT    NOT NULL,
  price_cents INTEGER NOT NULL,
  is_active   INTEGER NOT NULL DEFAULT 0 CHECK (is_active IN (0,1)),
  created_at  TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
) STRICT;                                  -- 3.37.0+
```

**`STRICT` is itself a constraint:** it forbids `DECIMAL` and `VARCHAR` spellings entirely, so adopting it
means rewriting the DDL, and `ALTER TABLE` cannot add `STRICT` to an existing table — you must recreate
and copy. `STRICT` still does not enforce length.

**Fine when:** a single ORM is the only writer and already enforces types, and the database is never
written by hand or by another service.

---

## `INT PRIMARY KEY` is not `INTEGER PRIMARY KEY`

Only the exact type name `INTEGER` makes a column an alias for the rowid. `INT PRIMARY KEY` is an
ordinary column with a unique index — no auto-assignment, and inserts that omit it store NULL.

Separately, a documented SQLite quirk means **primary key columns accept NULL** unless the column is
`INTEGER PRIMARY KEY`, declared `NOT NULL`, or the table is `WITHOUT ROWID` or `STRICT`. Multiple NULL
primary key rows can coexist.

Text and UUID primary keys are legitimate — they just need explicit `NOT NULL`.

---

## Indexes

**SQLite uses at most one index per table per query for ANDed terms.** It explicitly does not implement
AND-by-intersect, so two single-column indexes never combine — you need one composite index.

```sql
CREATE INDEX idx_f_s ON FruitsForSale(fruit, state, price);   -- one composite
ANALYZE;                        -- after bulk load or in deploy, never per request
```

**OR-connected terms are the exception** — SQLite can union results from several indexes there, so
multiple single-column indexes are correct for `WHERE a = ? OR b = ?`.

Without `ANALYZE`, `sqlite_stat1` is empty and the choice between candidate indexes is essentially
arbitrary — which is how a query silently picks the useless one. `PRAGMA optimize` on connection close is
the modern, cheap alternative.

Other specifics:

- **No index on foreign key child columns is created automatically** — same as Postgres, unlike MySQL.
- **Btree only.** No GIN, no GiST, no hash, no partial-index equivalent gap — partial indexes *do* exist
  (3.8.0+), and expression indexes exist (3.9.0+).
- **The `LIKE` prefix optimisation** applies only if `case_sensitive_like` is off **and** the column is
  declared `COLLATE NOCASE`, the pattern has no leading wildcard, and the column has TEXT affinity. An
  anchored `LIKE` on a default-collation column is often still a scan. Use `GLOB` instead.

---

## Migrations: the 12-step rebuild

SQLite has no `ALTER` for dropping a column (before 3.35.0), changing a type, or adding/removing a
constraint. The ORM silently performs the 12-step rebuild instead: create a new table, copy every row,
drop the original, rename. It is O(table size) — and if `PRAGMA foreign_keys` is ON during the rebuild,
the `DROP TABLE` step can cascade-delete rows in child tables.

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

**Native cheap `ALTER` does exist for:** `RENAME TABLE`, `ADD COLUMN`, `RENAME COLUMN` (3.25.0+),
`DROP COLUMN` (3.35.0+), `SET`/`DROP NOT NULL` (3.53.0+). **Do not flag those.**

`ADD COLUMN` legitimately cannot carry `UNIQUE`/`PRIMARY KEY`, a `CURRENT_TIMESTAMP` or
parenthesised-expression default, or `NOT NULL` without a non-null constant default. Those errors are
SQLite working as designed.

On a single-user development database the rebuild cost is irrelevant.

---

## Reading `EXPLAIN QUERY PLAN`

```
SCAN orders                                        -- bad: every row
SEARCH orders USING INDEX idx_user (user_id=?)     -- good: a subset
SEARCH orders USING INTEGER PRIMARY KEY (rowid=?)  -- best
```

SQLite gives no cost numbers at all, so this one word is the entire diagnosis. In a join, an inner-loop
`SCAN` runs once per outer row.

`USING AUTOMATIC INDEX` is SQLite telling you a permanent index is missing — it built a transient one at
query time and pays that cost on every execution. Always a finding.

`USING COVERING INDEX` is the good case — never flag it.

The output format is documented as unstable: 3.36.0 dropped the leading table numbers and the
`TABLE`/`SUBQUERY` keywords, so older tutorials show `SCAN TABLE t1` where current versions print
`SCAN t1`.

Full guide: [reading-plans.md](../references/reading-plans.md).

---

## Concurrency and deployment

SQLite is single-writer. The one setting that matters most for a web application:

```sql
PRAGMA journal_mode = WAL;      -- readers no longer block the writer
PRAGMA busy_timeout = 5000;     -- wait rather than failing instantly on SQLITE_BUSY
```

Without `busy_timeout`, concurrent writes surface as immediate `SQLITE_BUSY` errors rather than a short
wait — which reads as random 500s under load.

`journal_mode` is persistent (stored in the database file); `busy_timeout` and `foreign_keys` are
per-connection and must be set every time.

**Turso/libSQL and D1** are SQLite-compatible but network-attached, so round-trip latency dominates and
the "one index per table per query" rule matters more, not less. D1 in particular has no persistent
connection to set pragmas on — check the driver's documented defaults rather than assuming.

---

## Grep list

```bash
rg -in 'REFERENCES|FOREIGN KEY' --glob '*.sql' --glob '**/migrations/**'
rg -i 'foreign_keys\s*=\s*(ON|1|true)' --glob '**/*.{js,ts,py,rb,go,rs}'   # absence is the finding
rg -in '\b(INT|BIGINT|SMALLINT|TINYINT)\s+PRIMARY\s+KEY' --glob '*.sql'
rg -in '\b(decimal|numeric|boolean|bool|datetime|varchar|string|uuid)\s*\(' --glob '*.sql'
rg -in '\bSTRICT\b|WITHOUT ROWID' --glob '*.sql'
rg -in '\banalyze\b|pragma\s+optimize|journal_mode|busy_timeout' --glob '**/*.{sql,ts,js,py}'
rg -in 'create index' --glob '*.sql'        # multiple single-column indexes on one table
rg -in 'drop column|alter column|add constraint' --glob '**/migrations/**'   # triggers a rebuild
rg -n 'LIKE' --glob '*.{sql,ts,py}'
```

**Runtime checks worth asking the user to run:**

```sql
PRAGMA foreign_keys;          -- 0 means nothing is enforced
PRAGMA foreign_key_check;     -- lists existing orphans
SELECT * FROM sqlite_stat1;   -- empty means ANALYZE has never run
PRAGMA journal_mode;          -- 'delete' on a web app is a finding
```
