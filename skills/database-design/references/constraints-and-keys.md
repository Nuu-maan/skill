# Constraints and keys

The rules the database enforces for you. Load when reviewing primary keys, foreign keys, uniqueness,
defaults, or generated columns.

The theme of this file: **a constraint that exists in the schema file is not necessarily a constraint
the server enforces.** Several engines parse constraint syntax and ignore it. Check the version and
the connection setup, not just the DDL.

Every entry has a **Fine when** line. Read it before reporting.

## Contents

- SQLite foreign keys are off by default
- SQLite `INT PRIMARY KEY` is not `INTEGER PRIMARY KEY`
- MySQL `CHECK` before 8.0.16
- MyISAM silently discards foreign keys
- `UNIQUE` on a nullable column
- MongoDB unique indexes and missing fields
- Mongoose `unique: true` is not a validator
- Prisma `relationMode = "prisma"`
- Django and SQLAlchemy cascade in Python, not SQL
- Unindexed foreign key columns
- Random UUID as a clustered primary key
- Tables with no primary key
- `GENERATED ALWAYS AS` without `STORED`
- `SET NOT NULL` and `ADD CHECK` lock the table

---

## SQLite foreign keys are off by default

**Signal:** The schema declares `REFERENCES` / `FOREIGN KEY`, but no application code issues
`PRAGMA foreign_keys = ON`. Search the connection setup, not the schema.

**Why it breaks:** SQLite parses and stores foreign key definitions but does not enforce them unless
the pragma is set — and the setting is **per connection**, not per database. Every pooled connection,
every `sqlite3` CLI session, and every new worker starts at 0. The schema looks correct while orphan
rows accumulate silently and `ON DELETE CASCADE` never fires.

**Fix:** run it immediately after opening every connection.

```sql
PRAGMA foreign_keys = ON;
PRAGMA foreign_key_check;     -- then audit what already leaked through
```

```js
db.pragma('foreign_keys = ON')                     // better-sqlite3
```

```python
@event.listens_for(Engine, 'connect')              # SQLAlchemy
def _fk_on(conn, _): conn.execute('PRAGMA foreign_keys=ON')
```

**Fine when:** the driver does it for you. Django's SQLite backend, the Rails sqlite3 adapter,
`bun:sqlite`, and Prisma all set it per connection. If one of those is in use and nothing overrides
it, there is no finding.

Also flag `PRAGMA foreign_keys` issued **inside an open transaction** — it is a silent no-op there.

**Find it:** `REFERENCES` present in migrations, and zero hits for
`rg -i "foreign_keys\s*=\s*(ON|1|true)"` across the repo.

---

## SQLite `INT PRIMARY KEY` is not `INTEGER PRIMARY KEY`

**Signal:** `id INT PRIMARY KEY` or `id BIGINT PRIMARY KEY` in a SQLite table, or any `PRIMARY KEY`
column without `NOT NULL` on a rowid table.

**Why it breaks:** Only the exact type name `INTEGER` makes a column an alias for the rowid.
`INT PRIMARY KEY` is an ordinary column with a unique index — no auto-assignment, and inserts that
omit it store NULL. Separately, a long-standing documented SQLite quirk means PRIMARY KEY columns
accept NULL unless the column is `INTEGER PRIMARY KEY`, declared `NOT NULL`, or the table is
`WITHOUT ROWID` or `STRICT`. Multiple NULL primary key rows can coexist.

**Fix:**

```sql
CREATE TABLE users (
  id    INTEGER PRIMARY KEY,        -- exactly INTEGER
  email TEXT NOT NULL UNIQUE
) STRICT;                           -- STRICT also forces PK NOT NULL
```

For a non-integer key: `id TEXT NOT NULL PRIMARY KEY`.

**Fine when:** the column is also declared `NOT NULL`, or the table ends in `WITHOUT ROWID` or
`STRICT` (3.37.0+). A composite `PRIMARY KEY (a, b)` on a STRICT table is fine. Text and UUID primary
keys are legitimate in SQLite — they just need explicit `NOT NULL`.

---

## MySQL `CHECK` before 8.0.16

**Signal:** `CHECK (...)` constraints together with any sign the target is MySQL 5.7 or earlier —
`mysql:5.7` in `docker-compose.yml`, `DEFAULT CHARSET=utf8` dumps, an RDS 5.7 parameter group.

**Why it breaks:** MySQL parsed and **ignored** CHECK constraint syntax until 8.0.16. On 5.7 the
constraint exists in the DDL, the migration applies cleanly, and nothing is ever validated — the
schema reads as safe while invalid data lands. MariaDB enforces CHECK from 10.2.1, so a schema tested
on MariaDB regresses when deployed to MySQL 5.7.

**Fix:** verify the server actually enforces it.

```sql
SELECT VERSION();
SELECT CONSTRAINT_NAME, CHECK_CLAUSE, ENFORCED
  FROM information_schema.CHECK_CONSTRAINTS;      -- empty on 5.7
```

On 5.7 the only real options are `BEFORE INSERT`/`UPDATE` triggers, an `ENUM`/`SET` column, or
application validation.

**Fine when:** MySQL 8.0.16+ or MariaDB 10.2.1+, which covers most new work.

Also flag an explicit `NOT ENFORCED` on any CHECK — that is a deliberately disabled constraint. Note
that MySQL rejects CHECK expressions referencing `AUTO_INCREMENT` columns, other tables, subqueries,
or non-deterministic functions such as `NOW()`; a schema using those fails to create on 8.0.16+
rather than being ignored.

---

## MyISAM silently discards foreign keys

**Signal:** `ENGINE=MyISAM` (or `MEMORY`, `ARCHIVE`) on a `CREATE TABLE` that also contains a
`FOREIGN KEY` clause.

**Why it breaks:** Only InnoDB and NDB support foreign keys in MySQL. MyISAM parses the syntax and
ignores it entirely — no error, no constraint, no cascade. This shows up in old dumps and in
AI-generated schemas copied from pre-5.5 examples. MyISAM also has no transactions, so a partially
failed multi-table write leaves permanent inconsistency.

**Fix:**

```sql
ALTER TABLE orders ENGINE=InnoDB;

-- then re-add the FKs MyISAM discarded
ALTER TABLE orders
  ADD CONSTRAINT fk_orders_user
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT;

-- audit what actually exists
SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES
 WHERE TABLE_SCHEMA = DATABASE() AND ENGINE <> 'InnoDB';
```

**Fine when:** a read-only lookup table with no relationships and no transactional writes. It is only
a finding when FK clauses, transactional writes, or replication-critical data are involved. MySQL 8.0
stores system tables in InnoDB regardless, so an old dump mentioning MyISAM for `mysql.*` is not your
schema.

---

## `UNIQUE` on a nullable column

**Signal:** `email TEXT UNIQUE` with no `NOT NULL`. Or the soft-delete form
`UNIQUE (email, deleted_at)` where `deleted_at` is NULL for live rows.

**Why it breaks:** In PostgreSQL, MySQL, and SQLite, NULLs are not equal to each other, so a unique
index permits unlimited NULL rows — and a composite unique key containing a NULL column enforces
nothing for those rows. The soft-delete variant is the classic failure: every live row has
`deleted_at IS NULL`, the constraint never fires, and duplicate emails accumulate.

**Fix:**

```sql
-- Postgres 15+: make NULLs collide
ALTER TABLE users ADD CONSTRAINT users_email_key
  UNIQUE NULLS NOT DISTINCT (email);

-- any Postgres version, soft-delete safe
CREATE UNIQUE INDEX users_email_live
  ON users (email) WHERE deleted_at IS NULL;
```

MySQL and SQLite have no `NULLS NOT DISTINCT` — use `NOT NULL` plus a sentinel, or a generated column.

**Fine when:** many NULLs are deliberate. An optional `external_ref` that must be unique *when
present* is correctly modelled as a nullable UNIQUE column. Only flag it when the column is
business-required, or when it participates in a soft-delete composite key.

---

## MongoDB unique indexes and missing fields

**Signal:** `createIndex({ field: 1 }, { unique: true })`, or Mongoose
`field: { type: String, unique: true }`, where the field is not `required` and has no default.

**Why it breaks:** MongoDB indexes a missing field as null, so a unique single-field index permits
exactly **one** document without that field. The first document omitting it inserts fine; the second
fails with `E11000 duplicate key on null`. This is the reverse of SQL behaviour, and it surfaces in
production rather than tests, because tests usually set the field.

**Fix:**

```js
db.users.createIndex(
  { email: 1 },
  { unique: true, partialFilterExpression: { email: { $type: "string" } } }
);
```

**Fine when:** the field is `required: true` with no possibility of absence, or the index already
carries `sparse: true` or a `partialFilterExpression`.

In sharded collections a unique index is only enforceable on the shard key or a compound index
prefixed by it, and `_id` uniqueness is enforced per shard — not cluster-wide — when `_id` is not the
shard key.

---

## Mongoose `unique: true` is not a validator

**Signal:** `unique: true` as the only uniqueness enforcement, with no awaited `Model.init()` or
`syncIndexes()`, or with `autoIndex: false` in the connection options.

**Why it breaks:** Mongoose documents that `unique: true` is **not a validator** — it is shorthand for
creating a MongoDB unique index. If the index does not yet exist, duplicate saves succeed with no
error. Mongoose builds indexes in the background at startup, so the guarantee is absent during the
first seconds of a deploy, and absent entirely whenever `autoIndex` is disabled or the collection
already holds duplicates that block the build.

**Fix:**

```js
await User.init();          // or User.syncIndexes() — before serving traffic

try { await user.save(); }
catch (e) { if (e.code === 11000) return res.status(409).send('taken'); }
```

The duplicate-key error is a driver error, not a `ValidationError`, so it needs its own handling.

**Fine when:** indexes are created by an explicit migration step, an infrastructure script, or an
awaited `syncIndexes()` on boot — which is Mongoose's own production recommendation. `autoIndex:
false` is correct practice in production; the finding is only real when nothing else creates the
index.

---

## Prisma `relationMode = "prisma"`

**Signal:** `relationMode = "prisma"` in the `datasource` block while the provider is `postgresql`,
`mysql`, or `sqlite`. Also the legacy `previewFeatures = ["referentialIntegrity"]`.

**Why it breaks:** Prisma emits **no foreign keys in the database at all**; referential integrity and
referential actions are emulated by the client with extra queries. Anything writing outside Prisma — a
`psql` session, a job in another language, a data import — creates orphans freely, and cascade deletes
cost N extra round trips. Prisma also stops creating relation-field indexes, so you must add every one
by hand.

**Fix:**

```prisma
datasource db {
  provider     = "postgresql"
  url          = env("DATABASE_URL")
  relationMode = "foreignKeys"     // the default; delete the line
}
```

If the mode must stay, index every relation scalar:

```prisma
model Post {
  authorId Int
  author   User @relation(fields: [authorId], references: [id])
  @@index([authorId])
}
```

**Fine when:** running on PlanetScale or Vitess, or a serverless MySQL provider that forbids foreign
keys — and it is the only option for MongoDB. Only a finding when the database does support foreign
keys and nothing forces the mode. If it is genuinely required, **the missing `@@index` on relation
scalars is the real defect to report.**

---

## Django and SQLAlchemy cascade in Python, not SQL

**Signal:** Django `on_delete=models.CASCADE` / `SET_NULL`. SQLAlchemy relationships with
`cascade="all, delete-orphan"` but no `ondelete="CASCADE"` on the `ForeignKey` and no
`passive_deletes=True`.

**Why it breaks:** Django explicitly documents that `on_delete` does **not** create an SQL constraint
— it emulates the behaviour in Python by loading and deleting related objects. SQLAlchemy's ORM
cascade is likewise Python-side. So deleting a parent via raw SQL, a `TRUNCATE`, or another service
leaves orphans, and deleting a parent with many children issues thousands of individual statements.

**Fix:** keep the ORM behaviour and add the database rule underneath it.

```python
# SQLAlchemy
ForeignKey('post.id', ondelete='CASCADE')     # + passive_deletes=True on the relationship
```

```sql
-- Django: add the DB-level action in a RunSQL migration
ALTER TABLE app_comment DROP CONSTRAINT app_comment_post_id_fkey,
  ADD CONSTRAINT app_comment_post_id_fkey
  FOREIGN KEY (post_id) REFERENCES app_post(id) ON DELETE CASCADE;
```

**Fine when:** Python-side cascade is intentional — it fires `pre_delete`/`post_delete` signals,
cleans up uploaded files, and handles polymorphic relations the database cannot express. It is only a
defect when other writers touch the database directly, or when child counts make per-row deletes a
performance problem.

`db_constraint=False` in Django is a separate and more severe finding: there is no foreign key at all.

---

## Unindexed foreign key columns

**Signal:** `user_id UUID REFERENCES users(id)` in PostgreSQL or SQLite with no
`CREATE INDEX ... (user_id)`.

**Why it breaks:** Neither PostgreSQL nor SQLite indexes the referencing (child) column
automatically — **only MySQL does.** Without it, every parent `DELETE` or key `UPDATE` scans the whole
child table to find referencing rows, and in PostgreSQL that scan happens while holding locks. On a
few million child rows one parent delete can take minutes and block writers.

**Fix:**

```sql
CREATE INDEX CONCURRENTLY idx_posts_user_id ON posts (user_id);
```

Find every unindexed foreign key in Postgres:

```sql
SELECT c.conrelid::regclass, c.conname FROM pg_constraint c
 WHERE c.contype = 'f' AND NOT EXISTS (
   SELECT 1 FROM pg_index i WHERE i.indrelid = c.conrelid
     AND (i.indkey::smallint[])[0:array_length(c.conkey,1)-1] @> c.conkey);
```

**Fine when:** the column is already the leading column of a composite index or the primary key, or
the parent table's rows are never deleted or re-keyed (immutable lookup tables). A separate index is
wasteful on a child table of a few thousand rows that is never joined by that column.

In Django, `ForeignKey` sets `db_index=True` by default — there, the finding is only when
`db_index=False` is set explicitly.

---

## Random UUID as a clustered primary key

**Signal:** `id CHAR(36) PRIMARY KEY`, `id BINARY(16) PRIMARY KEY DEFAULT (UUID_TO_BIN(UUID()))`, or
Prisma `@id @default(uuid())` / `@default(cuid())` against a MySQL provider.

**Why it breaks:** InnoDB stores the table itself in primary key order, so random v4 UUID inserts land
at random offsets and cause constant B-tree page splits and poor buffer-pool locality. Every secondary
index also stores the full primary key, so a 36-byte key inflates every index on the table. Write
throughput degrades non-linearly once the index exceeds RAM.

**Fix:**

```sql
id BINARY(16) NOT NULL PRIMARY KEY
   DEFAULT (UUID_TO_BIN(UUID(), 1))     -- swap_flag=1, MySQL 8.0+
```

`swap_flag=1` swaps time-low and time-high so the slow-varying prefix leads, restoring insert
locality. Or use UUIDv7. Or keep a `BIGINT AUTO_INCREMENT` primary key and add a separate `UNIQUE`
uuid column for external URLs.

`UUID_TO_BIN(x)` **without** the second argument is itself the finding.

**Fine when:** on PostgreSQL, which does not cluster on the primary key — there the cost is index
bloat and WAL churn rather than table reorganisation, and it is acceptable at moderate scale. Also
fine well under ~100 inserts/sec, or when the whole index fits comfortably in the buffer pool.

---

## Tables with no primary key

**Signal:** A `CREATE TABLE` with no `PRIMARY KEY` and no `UNIQUE ... NOT NULL` index — usually a join
table, an events table, or an audit log.

**Why it breaks:** InnoDB falls back to a hidden 6-byte row ID you cannot query, index, or reference.
Row-based replication and most CDC tools (Debezium, Vitess, DMS) then full-scan the table to locate
each changed row, and replica lag explodes. In PostgreSQL there is no hidden key at all, so logical
replication of `UPDATE`/`DELETE` fails outright without a replica identity.

**Fix:**

```sql
ALTER TABLE post_tags ADD PRIMARY KEY (post_id, tag_id);   -- natural composite key
ALTER TABLE events ADD COLUMN id BIGINT AUTO_INCREMENT PRIMARY KEY;
ALTER TABLE events REPLICA IDENTITY FULL;                  -- Postgres fallback
```

**Fine when:** a genuinely append-only table that is never replicated, never updated, and never
row-referenced.

A composite `PRIMARY KEY (a, b)` only supports lookups by `a` or by `(a, b)` — queries filtering by
`b` alone still need their own index. That is the most common follow-on miss in join tables.

---

## `GENERATED ALWAYS AS` without `STORED`

**Signal:** `GENERATED ALWAYS AS (expr)` with no `STORED` or `VIRTUAL` keyword.

**Why it breaks:** The unqualified default differs by engine and changed in Postgres. PostgreSQL 12–17
accepted only `STORED`; PostgreSQL 18 added virtual generated columns and made `VIRTUAL` the default.
MySQL 5.7+ and SQLite 3.31+ have always defaulted to `VIRTUAL`. So a schema written against PG 12–17
and re-run on PG 18 silently produces a non-stored column — recomputed on every read — and **PG 18
does not support indexes on virtual generated columns**, so an index you intended will not exist.

**Fix:** always be explicit.

```sql
price_cents INT  GENERATED ALWAYS AS (price * 100)  STORED,
email_lower TEXT GENERATED ALWAYS AS (lower(email)) STORED;

CREATE UNIQUE INDEX ON users (email_lower);    -- only STORED is indexable in PG 18
```

**Fine when:** the expression is cheap, the column is rarely read, and it is never indexed — `VIRTUAL`
costs zero storage. This is only a defect when the schema is version-portable or an index depends on
the column.

PG 18 virtual columns additionally cannot use user-defined functions or types, so an existing `STORED`
column may not convert cleanly. SQLite's `ALTER TABLE ADD COLUMN` cannot add a `STORED` generated
column at all.

---

## `SET NOT NULL` and `ADD CHECK` lock the table

**Signal:** A migration running `ALTER TABLE ... ALTER COLUMN x SET NOT NULL` or
`ADD CONSTRAINT ... CHECK (...)` in one step on PostgreSQL, on a table that is not brand new.

**Why it breaks:** Both take an `ACCESS EXCLUSIVE` lock and scan the whole table to validate. On a
large table this blocks every read and write for the duration, and without a `lock_timeout` it queues
every subsequent query behind it.

**Fix:** two phases — a brief lock, then a lock-free scan.

```sql
SET lock_timeout = '3s';

ALTER TABLE t ADD CONSTRAINT t_x_nn CHECK (x IS NOT NULL) NOT VALID;
ALTER TABLE t VALIDATE CONSTRAINT t_x_nn;    -- SHARE UPDATE EXCLUSIVE only
ALTER TABLE t ALTER COLUMN x SET NOT NULL;   -- PG12+ reuses the CHECK, no rescan
```

PostgreSQL 18+ accepts `NOT VALID` on `NOT NULL` constraints directly.

**Fine when:** the table was created in the same migration, or is small enough (roughly under 100k
rows) that the scan is sub-second. The two-step is only worth the extra migration on a large table or
where the deploy cannot tolerate a write pause.

**Related:** PostgreSQL only accepts `DEFERRABLE` on `UNIQUE`, `PRIMARY KEY`, `EXCLUDE`, and
`REFERENCES` — never on `CHECK` or `NOT NULL`. A deferrable unique constraint also cannot be used as
an `ON CONFLICT` arbiter. MySQL has no deferrable constraints at all; SQLite supports
`DEFERRABLE INITIALLY DEFERRED` on foreign keys only.

---

## Sources

SQLite: [foreignkeys](https://www.sqlite.org/foreignkeys.html),
[lang_createtable](https://www.sqlite.org/lang_createtable.html).
PostgreSQL: [sql-createtable](https://www.postgresql.org/docs/current/sql-createtable.html),
[release-18](https://www.postgresql.org/docs/18/release-18.html).
MySQL: [check-constraints](https://dev.mysql.com/doc/refman/8.4/en/create-table-check-constraints.html),
[foreign-keys](https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html),
[innodb-index-types](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html).
MongoDB: [index-unique](https://www.mongodb.com/docs/manual/core/index-unique/).
Prisma: [relation-mode](https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/relation-mode).
