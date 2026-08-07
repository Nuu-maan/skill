# MySQL and MariaDB

Load when the project uses MySQL, MariaDB, PlanetScale, or Vitess. One engine file per review.

MySQL's defining trait for this skill: **it accepts things silently.** Constraints that are parsed and
ignored, comparisons that coerce rather than error, DDL that quietly rewrites a whole table. Most
findings here are things that look fine in the schema file and are not enforced by the server.

## Contents

- The five most common defects
- Version matters — check it first
- Things MySQL accepts silently
- DDL: state the algorithm or find out the hard way
- InnoDB specifics that change the schema
- Reading `EXPLAIN`
- PlanetScale and Vitess
- Grep list

---

## The five most common defects in an AI-generated MySQL schema

1. **`CHARSET=utf8`**, which is a 3-byte encoding that cannot store emoji. `utf8mb4` is the real one.
2. **`ALTER TABLE` with no `ALGORITHM=`**, so MySQL silently picks `COPY` and blocks writes for the
   duration. → [migrations.md](../references/migrations.md)
3. **`CHECK` constraints targeting MySQL 5.7**, where they were parsed and ignored entirely.
4. **A random UUID as the primary key**, which is the clustered index — so random inserts scatter the
   *table*, not just an index. → [constraints-and-keys.md](../references/constraints-and-keys.md)
5. **`TIMESTAMP` on a column that must hold dates past 2038.**

## Version matters — check it first

`SELECT VERSION();` and look for the image tag in `docker-compose.yml` or the RDS parameter group.

| Version | What changed |
| :--- | :--- |
| 5.6 | Most `ADD INDEX` became online |
| 5.7.9 | `DYNAMIC` row format became the default (index key limit 3072, not 767) |
| 8.0.12 | `ALGORITHM=INSTANT` for `ADD COLUMN` |
| 8.0.13 | Functional (expression) key parts — before this there are no expression indexes at all |
| **8.0.16** | **`CHECK` constraints began being enforced.** Before: parsed and ignored |
| 8.0.18 | `EXPLAIN ANALYZE` |
| 8.0.29 | Instant `ADD COLUMN` in any position; instant `DROP COLUMN` |
| MariaDB 10.2.1 | `CHECK` enforced (earlier than MySQL) |
| MariaDB 11.6 | Default charset finally moved from `latin1` to `utf8mb4` |

**A schema tested on MariaDB regresses when deployed to MySQL 5.7** — MariaDB enforced `CHECK` seven
years earlier.

---

## Things MySQL accepts silently

- **`CHECK` before 8.0.16** — the constraint exists in the DDL, the migration applies, nothing is ever
  validated. Confirm with `SELECT * FROM information_schema.CHECK_CONSTRAINTS;` — empty on 5.7. Also
  flag an explicit `NOT ENFORCED`.
- **`ENGINE=MyISAM` with a `FOREIGN KEY` clause** — MyISAM parses foreign keys and discards them. No
  error, no constraint, no cascade. MyISAM also has no transactions.
- **`BOOLEAN`** is an alias for `TINYINT(1)`, which accepts −128 to 127. `UPDATE t SET is_active = 2`
  succeeds, and then `WHERE is_active = TRUE` and `WHERE is_active` disagree.
- **A string column compared to a number** — `WHERE varchar_col = 0` matches every row that converts to
  0, including `'apple'`. The index is dead and the result set is the whole table.
- **`ENUM('1','2','3')` compared to the integer `1`** matches by **ordinal**, not value. `ORDER BY` on an
  enum sorts by declaration order.
- **`USING HASH`** on InnoDB is accepted and ignored — InnoDB has BTREE, FULLTEXT, and SPATIAL only.
- **`CREATE INDEX ... WHERE`** does not exist. MySQL has **no partial or filtered indexes**, and
  AI-written MySQL schemas hallucinate them regularly. That is a migration failure, not a slow query.

---

## DDL: state the algorithm or find out the hard way

```sql
ALTER TABLE orders ADD COLUMN note TEXT, ALGORITHM=INSTANT;
ALTER TABLE orders ADD INDEX idx_x (x), ALGORITHM=INPLACE, LOCK=NONE;
```

If it errors with 1845, the change is **not** online — and now you know before production does. Route it
through `gh-ost` or `pt-online-schema-change` instead.

Operations that fall back to `ALGORITHM=COPY`: any column type change, shrinking a VARCHAR, widening a
VARCHAR **across the 255-byte length-prefix boundary** (63 → 64 characters under utf8mb4, not 255 → 256),
`DROP PRIMARY KEY`, adding a `STORED` generated column, most partitioning changes.

Instant add/drop is capped at **64 row versions per table** — past that, `ERROR 4092` and a rebuild.

**Every DDL statement implicitly commits.** If statement 3 of 5 fails, statements 1 and 2 stay applied
while the tool records the migration as failed, and re-running dies on "column already exists." One DDL
statement per migration file, each idempotent. (Postgres and SQLite have transactional DDL and do not
have this problem — do not flag multi-statement migrations there.)

---

## InnoDB specifics that change the schema

- **The table is stored in primary key order.** A random UUIDv4 primary key causes constant page splits
  and poor buffer-pool locality. Use `BINARY(16)` with `UUID_TO_BIN(UUID(), 1)` — the `1` swaps time-low
  and time-high so the slow-varying prefix leads — or UUIDv7, or a `BIGINT AUTO_INCREMENT` PK plus a
  separate unique uuid column for URLs.
- **Every secondary index silently appends the primary key columns.** So `INDEX (d)` on a table with
  `PRIMARY KEY (i1, i2)` is physically `(d, i1, i2)` — which makes an explicit `(d, i1)` index redundant,
  and makes a wide primary key inflate *every* index on the table.
- **InnoDB auto-creates an index on the child side of a foreign key** if none exists with the FK columns
  leftmost. This is the one place MySQL is friendlier than Postgres — **do not report unindexed foreign
  keys here.**
- **Index key limit is 3072 bytes** (DYNAMIC/COMPRESSED). Under utf8mb4 a `VARCHAR(255)` costs 1020
  bytes, so three of them in one composite key fails outright. Max 16 columns per index.
- **A table with no primary key** falls back to a hidden 6-byte row id you cannot query or reference, and
  CDC tools then full-scan to locate each changed row.

---

## Reading `EXPLAIN`

The `type` column, worst to best: `ALL` < `index` < `range` < `ref` < `eq_ref` < `const`.

**The trap is `index`** — that is a full scan *of the index*, not a lookup, and only marginally better
than `ALL`. It is fine only when `Extra` also says `Using index`.

`Using filesort` does **not** mean disk. It is MySQL's name for any sort not served by an index, done in
memory for small sets. `Using where` alone is normal. The real finding is
`type: ALL` + `Using where` + `key: NULL`.

Plain `EXPLAIN` is estimates only — `rows` comes from sampled index dives and can be off by an order of
magnitude. Ask for `EXPLAIN ANALYZE` (8.0.18+) before making a performance claim. Full guide:
[reading-plans.md](../references/reading-plans.md).

---

## PlanetScale and Vitess

Foreign keys are unavailable or restricted, so `relationMode = "prisma"` (or the equivalent) is
**correct and required** there — not a defect. But it means the database creates no index for relations
either, so **every relation scalar needs an explicit index**, and referential integrity is enforced only
in application code. Anything writing outside the ORM creates orphans.

Online schema changes are handled by the platform, so the `ALGORITHM=` advice above is less relevant; the
`gh-ost` guidance does not apply.

---

## Grep list

```bash
rg -in 'charset\s*=?\s*.?utf8[^m]|CHARACTER SET utf8[^m]|COLLATE utf8_' --glob '*.sql' --glob '*.cnf' --glob '*.yml'
rg -in '^\s*alter table' --glob '**/migrations/**' --glob '*.sql'    # flag any lacking ALGORITHM=
rg -in '\bCHECK\s*\(|NOT\s+ENFORCED' --glob '*.sql' --glob '**/migrations/**'
rg -in 'mysql:5\.|mariadb:10\.[01]|5\.7' docker-compose*.yml Dockerfile* .github/workflows/*.yml
rg -in 'ENGINE\s*=\s*(MyISAM|MEMORY|ARCHIVE|CSV)' --glob '*.sql' --glob '*.dump'
rg -in 'CHAR\(36\).*PRIMARY KEY|UUID_TO_BIN\(UUID\(\)\)' --glob '*.sql'
rg -in '\bTIMESTAMP\b' --glob '*.sql'      # hardest on expire|end|valid_until|scheduled
rg -in '\b(bool|boolean|tinyint\(1\))\b' --glob '*.sql'   # then look for a CHECK
rg -in 'create index[^;]*\bwhere\b' --glob '*.sql'        # partial indexes do not exist here
rg -c -i '^\s*(alter|create|drop|rename) ' --glob '**/migrations/*.sql'   # >1 per file is the finding
```

**Redundant indexes** — MySQL has this built in:

```sql
SELECT * FROM sys.schema_redundant_indexes;
SELECT * FROM sys.schema_unused_indexes;
SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES
 WHERE TABLE_SCHEMA = DATABASE() AND ENGINE <> 'InnoDB';
```
