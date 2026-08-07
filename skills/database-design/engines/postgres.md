# PostgreSQL

Load when the project uses PostgreSQL, Supabase, Neon, RDS Postgres, or Cockroach. One engine file per
review.

Postgres is the most forgiving of the four engines about *correctness* and the least forgiving about
*locking*. Most serious findings here are one of two kinds: an index the planner cannot use, or a
migration that takes the site down.

## Contents

- The five most common defects
- Version matters — check it first
- Locking: the migration rules
- Indexes the planner cannot use
- Correctness traps
- Reading plans
- Supabase
- Grep list

---

## The five most common defects in an AI-generated Postgres schema

1. **No index on foreign key child columns.** Postgres indexes the parent's referenced key and **never**
   the child column. MySQL does; Postgres does not. → [indexes.md](../references/indexes.md)
2. **`timestamp` instead of `timestamptz`.** Same 8 bytes, and Postgres silently discards the offset on
   input. → [schema-and-types.md](../references/schema-and-types.md)
3. **`CREATE INDEX` with no `CONCURRENTLY`** in a migration against a live table.
   → [migrations.md](../references/migrations.md)
4. **No `lock_timeout`** anywhere in the migration path. One long-running `SELECT` turns a 1 ms DDL into
   a full outage.
5. **RLS enabled without `FORCE`**, while the app connects as the table owner — so the policy does
   nothing. → [modeling-patterns.md](../references/modeling-patterns.md)

## Version matters — check it first

`SHOW server_version;` Behaviour below is version-gated, and getting this wrong produces false findings.

| Version | What changed |
| :--- | :--- |
| 9.2 | Increasing a `varchar` length limit became catalog-only (no rewrite) |
| 11 | `ADD COLUMN` with a **non-volatile** default became metadata-only; `INCLUDE` on btree indexes |
| 12 | `SET NOT NULL` can reuse a validated `CHECK` instead of rescanning; generated columns (`STORED` only) |
| 14 | `CYCLE` clause on recursive CTEs |
| 15 | `UNIQUE NULLS NOT DISTINCT` |
| 18 | `uuidv7()`; **`VIRTUAL` became the default for generated columns**; btree skip scan; OR-clauses transformed to arrays; `BUFFERS` on by default in `EXPLAIN ANALYZE`; `NOT NULL` accepts `NOT VALID` |

The PG 18 generated-column change is the sharpest: a schema written for 12–17 that assumed `STORED` gets
a virtual column on 18, **and PG 18 cannot index virtual generated columns**, so an index you intended
silently will not exist.

---

## Locking: the migration rules

Postgres grants locks **FIFO**. An `ALTER TABLE` waiting behind one long `SELECT` blocks every
*subsequent* reader. This is why a metadata-only change can still cause an outage.

```sql
SET lock_timeout = '3s';     -- fail fast, then retry the migration
```

| Operation | Lock | Safe form |
| :--- | :--- | :--- |
| `CREATE INDEX` | blocks writes for the build | `CREATE INDEX CONCURRENTLY`, alone, outside a transaction |
| `SET NOT NULL` | `ACCESS EXCLUSIVE` + full scan | `ADD CHECK ... NOT VALID` → `VALIDATE` → `SET NOT NULL` |
| `ADD FOREIGN KEY` | `SHARE ROW EXCLUSIVE` on **both** tables | `... NOT VALID` then `VALIDATE CONSTRAINT` |
| `ADD CHECK` | `ACCESS EXCLUSIVE` + full scan | `NOT VALID` then `VALIDATE` |
| `ALTER COLUMN TYPE` | full rewrite unless binary-coercible | expand and contract |
| `ADD COLUMN ... DEFAULT` | metadata-only on 11+ **unless the default is volatile** | add nullable, backfill, then set the default |

`CONCURRENTLY` cannot run inside a transaction block, and most migration runners wrap every migration in
one. A failed concurrent build leaves an **INVALID** index that queries ignore while every write still
maintains it:

```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
```

Full detail and the per-framework opt-outs: [migrations.md](../references/migrations.md).

---

## Indexes the planner cannot use

- **A function on the column** — `WHERE lower(email) = $1` — needs an expression index, and the function
  must be `IMMUTABLE`.
- **Anchored `LIKE 'foo%'`** does not use a plain btree index unless the database is in the C locale or
  the index was built with `text_pattern_ops`.
- **Leading-wildcard `LIKE '%x%'`** needs `pg_trgm` + a GIN index. Nothing else will serve it.
- **A partial index** is only used when Postgres can *prove* the predicate is implied by the `WHERE`
  clause. It does not reason about equivalent rewrites — the query must state the predicate verbatim.
- **A `jsonb` path** — `data->>'email'` — needs an expression index on that exact expression. A GIN index
  on the column serves containment (`@>`), not `->>`.

Index types worth knowing: `gin` for jsonb and arrays and full text, `gist` for `ltree` and ranges,
`brin` only when values correlate with physical row order (append-only time series), `hash` only for `=`.

---

## Correctness traps

- **`NOT IN (SELECT ...)` over a nullable column returns zero rows.** One NULL makes the whole predicate
  NULL for every row. Use `NOT EXISTS`. This is the highest-severity query bug in the whole skill.
- **`money`** is exact but its output depends on `lc_monetary`, so a dump restored under a different
  locale can fail to load. Use `numeric`.
- **Arrays cannot have a foreign key on their elements.** `tag_ids integer[]` accumulates dangling ids
  forever, silently.
- **Enums cannot have a value removed**, and a value added inside a transaction cannot be used until it
  commits — which breaks single-transaction migration runners that add and backfill in one step.
- **Unquoted identifiers fold to lowercase.** A quoted `"userId"` column must be quoted everywhere,
  forever.
- **Tables with no primary key** cannot be logically replicated for `UPDATE`/`DELETE` without
  `REPLICA IDENTITY FULL` — there is no hidden row id as there is in InnoDB.

---

## Reading plans

`EXPLAIN (ANALYZE, BUFFERS)` — `BUFFERS` is automatic from 18.

The three numbers that matter most: `Rows Removed by Filter` (missing index), estimate-vs-actual
divergence (stale statistics), and `loops` (multiply before judging a node's cost). Autoanalyze only
fires after 50 rows + 10% of the table changes, so large tables are routinely stale:

```sql
SELECT relname, n_live_tup, n_mod_since_analyze, last_analyze, last_autoanalyze
  FROM pg_stat_user_tables ORDER BY n_mod_since_analyze DESC LIMIT 20;
```

Thresholds and the full node-by-node guide: [reading-plans.md](../references/reading-plans.md).

---

## Supabase

Supabase is Postgres, so everything above applies. Three things differ in practice:

- **The `authenticated` and `anon` roles are not table owners**, so the owner-bypasses-RLS trap usually
  does *not* apply — unlike a self-hosted setup where the app connects as the migration user. Check
  before flagging.
- **Policies live in the dashboard as well as in `supabase/migrations/`.** If the repo has no policy
  files, you cannot review authorization from files at all. Say so rather than assuming there are none.
- **PostgREST embedded selects** (`select=*,author(*)`) become joins. They obey the same index rules as
  any other join, and an unindexed foreign key hurts them the same way.

RLS *policy correctness* — who can read which rows — belongs to `backend-hardening`, not this skill. What
belongs here is whether RLS is inert (`ENABLE` without `FORCE`) and whether the indexes support the
policy predicates.

---

## Grep list

```bash
rg -in 'timestamp(\([0-9]\))?\s*(not null|default|,|\)|$)' --glob '**/migrations/**' --glob '*.sql'
rg -in 'create (unique )?index' --glob '**/migrations/**'      # then filter for CONCURRENTLY
rg -in 'lock_timeout|statement_timeout' --glob '**/migrations/**'   # absence is the finding
rg -in 'set not null|add constraint.*(check|foreign key)' --glob '**/migrations/**'
rg -in 'add column[^;]*default[^;]*(gen_random_uuid|uuid_generate|clock_timestamp|random\(\))'
rg -in 'alter column [a-z_"]+ (set data )?type|alter type .* add value' --glob '**/migrations/**'
rg -in 'row level security|create policy|force row level' --glob '**/migrations/**' --glob 'supabase/**'
rg -in '\b(int|integer|bigint|uuid|text)\s*\[\]|using (gin|gist|brin|hash)' --glob '*.sql'
rg -in 'not\s+in\s*\(\s*select' --glob '*.{sql,ts,js,py}'
rg -n 'REFERENCES' --glob '*.sql' --glob '**/migrations/**'    # each needs a child-side index
```

**The unindexed-foreign-key query** — run this one first if the user can give you output:

```sql
SELECT c.conrelid::regclass AS child, c.conname FROM pg_constraint c
 WHERE c.contype = 'f' AND NOT EXISTS (
   SELECT 1 FROM pg_index i WHERE i.indrelid = c.conrelid
     AND (i.indkey::smallint[])[0:array_length(c.conkey,1)-1] @> c.conkey);
```
