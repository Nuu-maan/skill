# Schema and types

Choosing the right column type, and the traps in each engine. Load when reviewing any column
definition.

Every entry has a **Fine when** line. Read it before reporting — most of these patterns have a
legitimate use, and a false positive costs the user more than a miss.

## Contents

- Money in a float
- Postgres `timestamp` vs `timestamptz`
- MySQL `TIMESTAMP` and the 2038 limit
- MySQL `utf8` is not UTF-8
- InnoDB index key length
- UUIDs stored as strings
- Random UUID primary keys
- Native enums
- JSON columns as schema avoidance
- SQLite type affinity
- Dates stored as strings
- Booleans in MySQL and SQLite
- Postgres arrays as relationships
- MySQL `VARCHAR` length changes

---

## Money in a float

**Signal:** A monetary column declared as a binary float — `price DOUBLE PRECISION`, `amount FLOAT`,
`total REAL`, Prisma `Float`, Mongoose `Number`.

**Why it breaks:** IEEE-754 binary floats cannot represent `0.1` or `19.99` exactly. Sums drift by
cents and `WHERE total = 19.99` misses rows. Invisible on test data, appears in production
reconciliation. Prisma `Float` maps to `double precision`; Mongoose `Number` becomes a BSON double.

**Fix:**

```sql
-- Postgres / MySQL
amount NUMERIC(12,2) NOT NULL        -- MySQL: DECIMAL(12,2)
-- SQLite has no exact decimal: store minor units
amount_cents INTEGER NOT NULL
```

```ts
amount Decimal @db.Decimal(12,2)           // Prisma
{ amount: Decimal128("19.99") }            // MongoDB / Mongoose Decimal128
```

**Fine when:** the quantity is genuinely approximate — sensor readings, ML scores, latitude and
longitude, percentages used only for display — or the column already stores integer minor units.

Postgres `money` is exact but its output format depends on `lc_monetary`, so it breaks across a
dump and restore under a different locale. Prefer `numeric`.

**Find it:** `rg -in '\b(float|double precision|double|real)\b'` over `*.sql`, `migrations/**`,
`*.prisma`, `models/**`, then check whether the column name matches
`price|amount|total|balance|cost|fee|salary|rate|tax`.

---

## Postgres `timestamp` vs `timestamptz`

**Signal:** `created_at timestamp` / `TIMESTAMP WITHOUT TIME ZONE`. Also plain Prisma `DateTime`,
whose default native type on PostgreSQL is `timestamp(3)` — without time zone — and Drizzle
`timestamp('created_at')` without `{ withTimezone: true }`.

**Why it breaks:** `timestamp` stores wall-clock digits with no instant attached, and PostgreSQL
silently discards any offset in the input. Two servers with different `TimeZone` settings write
different instants into the same column, and DST transitions duplicate or skip an hour.
`timestamptz` is the same 8 bytes and normalises everything to UTC.

**Fix:**

```sql
created_at timestamptz NOT NULL DEFAULT now()

-- migrating (rewrites the table; state the source zone explicitly)
ALTER TABLE t ALTER COLUMN created_at TYPE timestamptz
  USING created_at AT TIME ZONE 'UTC';
```

```ts
createdAt DateTime @default(now()) @db.Timestamptz(3)        // Prisma
timestamp('created_at', { withTimezone: true, mode: 'date' }) // Drizzle
```

**Fine when:** the value is genuinely floating wall-clock — a recurring 09:00 alarm, a store's
opening hours, a birth date-time, a scheduled local event. `date` alone is right for calendar dates.

`timestamptz` does not store the original offset either. If you need to round-trip the user's zone,
add a separate `tz text` column.

---

## MySQL `TIMESTAMP` and the 2038 limit

**Signal:** MySQL/MariaDB `TIMESTAMP` rather than `DATETIME`, especially on expiry, subscription-end,
or scheduled-for columns.

**Why it breaks:** `TIMESTAMP` has a hard range of `1970-01-01 00:00:01` UTC to
`2038-01-19 03:14:07` UTC. Anything past that is rejected or truncated, so a 20-year subscription end
date fails today. It also converts on write and read using the session `time_zone`, so a client
connecting with a different zone reads back a different wall-clock value.

**Fix:**

```sql
created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
expires_at DATETIME(6) NULL
```

```sql
SET time_zone = '+00:00';   -- or connectionTimeZone=UTC in the driver DSN
```

**Fine when:** the column only ever holds near-present values — `created_at` / `updated_at` on a
system with a planned life shorter than 2038 — where 4 bytes versus 8 matters and every client sets
the same zone.

`DATETIME` does **not** convert time zones, so it is only correct if the application always writes
UTC. A codebase that writes local time into `DATETIME` is a finding in its own right.

**Find it:** `rg -in '\bTIMESTAMP\b'` over `*.sql` and `migrations/**`. Flag hardest when the column
name matches `expire|end|valid_until|scheduled|due|renew`.

---

## MySQL `utf8` is not UTF-8

**Signal:** `CHARSET=utf8`, `CHARACTER SET utf8`, `COLLATE utf8_general_ci`, or a `my.cnf` / DSN with
`charset=utf8`.

**Why it breaks:** In MySQL, `utf8` is a deprecated alias for `utf8mb3`, a 3-byte-max encoding that
cannot store any character outside the BMP. Emoji and some CJK extension characters raise
`Incorrect string value` and are rejected — or silently truncated under non-strict SQL mode. MariaDB
defaulted to `latin1` through 11.4 and only switched to `utf8mb4` in 11.6.

**Fix:**

```sql
CREATE TABLE t (...) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

ALTER TABLE t CONVERT TO CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;          -- MariaDB: utf8mb4_uca1400_ai_ci
```

Set `charset=utf8mb4` in the driver too.

**Fine when:** the data is provably ASCII-only — country codes, hex digests, app-generated slugs.

Collation is a separate decision: `utf8mb4_0900_ai_ci` (the MySQL 8.0 default) is accent- and
case-insensitive; use `utf8mb4_bin` or `_as_cs` for exact matching. Changing collation on an indexed
column rebuilds the index.

**Find it:** `rg -in 'charset\s*=?\s*.?utf8[^m]|CHARACTER SET utf8[^m]|COLLATE utf8_'`. The absence
of any charset clause is also a signal — the table inherits the server default.

---

## InnoDB index key length

**Signal:** A composite index or unique key over several `VARCHAR(255)` columns in a `utf8mb4` table,
such as `UNIQUE KEY (email, tenant_slug, provider)`.

**Why it breaks:** InnoDB caps an index key at 3072 bytes for DYNAMIC and COMPRESSED row formats
(767 for the legacy REDUNDANT and COMPACT formats, and it scales down with `innodb_page_size` — 1536
at 8K, 768 at 4K). Under utf8mb4 each character reserves 4 bytes, so `VARCHAR(255)` costs 1020 bytes.
Three of them exceed 3072 and `CREATE INDEX` fails outright. Maximum 16 columns per index.

**Fix:**

```sql
-- size the column to what it actually holds
email VARCHAR(320), tenant_slug VARCHAR(63)

-- or index a prefix
UNIQUE KEY uk (email(191), tenant_slug, provider)

-- or index a hash of the wide value
email_hash BINARY(32) AS (UNHEX(SHA2(email,256))) STORED,
UNIQUE KEY (email_hash)
```

**Fine when:** a single `VARCHAR(255)` utf8mb4 column (1020 bytes), or in Postgres and SQLite, which
have no comparable limit.

Row format has been DYNAMIC by default since MySQL 5.7.9, so the 767-byte limit only applies to old
tables. Prisma's default `String` on MySQL is `varchar(191)` — exactly 767/4 — a legacy workaround,
not a modern requirement.

**Budget:** sum of `declared_length × bytes_per_char`, plus 2 per varchar, must be ≤ 3072.

---

## UUIDs stored as strings

**Signal:** UUID primary keys as `CHAR(36)`, `VARCHAR(36)`, or `TEXT` in MySQL/SQLite, or as
`text`/`varchar` in Postgres where a native `uuid` type exists.

**Why it breaks:** A 36-character UUID is 36 bytes — 144 reserved under utf8mb4 — against 16 bytes
binary. In InnoDB every secondary index embeds the full primary key, so a wide PK inflates every
index on the table, often 2–3× total index size. That is what stops fitting in the buffer pool.
String comparison is also slower than a 16-byte `memcmp`.

**Fix:**

```sql
-- Postgres
id uuid PRIMARY KEY DEFAULT gen_random_uuid()

-- MySQL 8.0+
id BINARY(16) PRIMARY KEY,
-- insert: UUID_TO_BIN(?, 1)   read: BIN_TO_UUID(id, 1)
-- swap_flag=1 reorders time-low/time-high so v1 UUIDs sort by time

-- SQLite
id BLOB PRIMARY KEY
```

**Fine when:** a small table (under ~1M rows, few secondary indexes) where reading IDs directly in a
shell and in logs is worth the bytes — `BINARY(16)` makes every ad-hoc query need a wrapper function.

In Postgres there is no tradeoff: the native `uuid` type is already 16 bytes *and* prints readably,
so `text` is simply wrong.

---

## Random UUID primary keys

**Signal:** `DEFAULT gen_random_uuid()`, `uuid_generate_v4()`, `UUID_TO_BIN(UUID())`, or
`crypto.randomUUID()` used as the primary key default on a high-insert table.

**Why it breaks:** UUIDv4 is uniformly random, so consecutive inserts land in random B-tree pages.
Once the index exceeds RAM every insert becomes a random read plus a page split, and the index bloats
as fill factor drops toward 50%. UUIDv7 puts a millisecond timestamp in the high bits, so inserts
append at the right edge like a sequence while staying unguessable.

**Fix:**

```sql
-- PostgreSQL 18+ has it natively
id uuid PRIMARY KEY DEFAULT uuidv7()

-- earlier Postgres / MySQL: generate v7 in the application
-- MySQL 8.0, from a v1 UUID:
id BINARY(16) DEFAULT (UUID_TO_BIN(UUID(), 1))
```

**Fine when:** below roughly a million rows, or on a low-insert table. Never raise this on a settings
or countries table.

There is a real security tradeoff: **UUIDv7 leaks the row's creation timestamp.** If IDs are exposed
publicly and creation time is sensitive — invites, private uploads — v4 is the correct choice. And
never switch an existing primary key type for this reason alone; the migration cost dwarfs the gain.

---

## Native enums

**Signal:** Postgres `CREATE TYPE status AS ENUM ('active','cancelled')`, a Prisma `enum`, or MySQL
`status ENUM('active','cancelled')`.

**Why it breaks:** In Postgres you can `ADD VALUE` and `RENAME VALUE`, but there is **no way to drop
one**. Removing a value means creating a new type, rewriting every column that uses it, and swapping.
`ALTER TYPE ... ADD VALUE` may run inside a transaction block, but the new value cannot be used until
that transaction commits — which breaks single-transaction migration runners that add a value and
backfill in one step.

**Fix:**

```sql
ALTER TYPE status ADD VALUE IF NOT EXISTS 'paused';
COMMIT;                      -- must commit before 'paused' is usable
UPDATE orders SET status = 'paused' WHERE ...;

-- or avoid the type entirely
status text NOT NULL REFERENCES order_status(code)
```

**Fine when:** the set is genuinely closed and rarely changes — weekday, currency side,
yes/no/unknown — where the 4-byte storage and the hard constraint are worth it. Prefer `text` plus a
`CHECK (status IN (...))`, or a lookup table, when the values are product-driven.

MySQL `ENUM` has a separate sharp edge: `ENUM('1','2','3')` compared against the integer `1` matches
by **ordinal**, and `ORDER BY` sorts by declaration order rather than alphabetically. Always declare
enum members as non-numeric strings.

---

## JSON columns as schema avoidance

**Signal:** A `jsonb`/`json`/`JSON` column named `data`, `meta`, `attributes`, `payload`, or
`settings` whose keys are read with fixed names — `WHERE data->>'email' = $1`, `user.meta.plan`, or a
Zod/TypeScript type that fully enumerates the object.

**Why it breaks:** If every key is known at build time, the JSON column is a schema in disguise: no
`NOT NULL`, no foreign keys, no type checks. The planner has no per-key statistics, so it falls back
to a fixed selectivity guess and picks bad plans. Filtering on `data->>'email'` cannot use a B-tree
index unless you create an expression index on that exact expression.

**Fix:**

```sql
-- promote known keys to real columns
ALTER TABLE users ADD COLUMN email text;
UPDATE users SET email = data->>'email';
ALTER TABLE users ALTER COLUMN email SET NOT NULL;

-- if it must stay JSON, index the actual predicate
CREATE INDEX ON users ((data->>'email'));
CREATE INDEX ON docs USING GIN (data jsonb_path_ops);   -- @> only
```

**Fine when:** the key set is genuinely dynamic or unbounded — user-defined fields, third-party
webhook payloads kept verbatim for replay, sparse per-tenant custom attributes, versioned document
snapshots. The tell for legitimate use is a dynamic key in the code rather than a fixed literal.

In Postgres prefer `jsonb` over `json`: `json` is stored as raw text, reparsed on every access, and
unindexable. But `jsonb` does not preserve key order or duplicate keys, so keep `json` for a signed
payload whose exact bytes matter. MariaDB's `JSON` is only a LONGTEXT alias and has none of jsonb's
indexing.

**Find it:** extract the key set actually used —
`rg -o "->>?'[a-z_]+'|json_extract\(\w+, *'\$\.[a-z_]+'" | sort -u`. Under ~10 distinct keys with no
dynamic key variable means schema avoidance.

---

## SQLite type affinity

**Signal:** A SQLite schema using types SQLite does not have: `price DECIMAL(10,2)`,
`is_active BOOLEAN`, `created_at DATETIME`, `name VARCHAR(50)`, `id UUID`.

**Why it breaks:** SQLite ignores the parenthesised size and maps the declared type to one of five
affinities by substring match. `DECIMAL(10,2)`, `BOOLEAN`, and `DATETIME` all get NUMERIC affinity —
so a money value is stored as a float — and `VARCHAR(50)` accepts a 10,000-character string without
error. Nothing rejects a wrong-typed value; a stray string in an integer column sorts before every
number.

**Fix:**

```sql
CREATE TABLE items (
  id          INTEGER PRIMARY KEY,
  name        TEXT    NOT NULL,
  price_cents INTEGER NOT NULL,
  is_active   INTEGER NOT NULL DEFAULT 0 CHECK (is_active IN (0,1)),
  created_at  TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
) STRICT;                    -- SQLite 3.37.0+
```

**Fine when:** a single ORM is the only writer and already enforces types, and the database is never
written by hand or by another service.

`STRICT` is itself a constraint: it forbids `DECIMAL` and `VARCHAR` spellings entirely, so adopting it
means rewriting the DDL, and `ALTER TABLE` cannot add `STRICT` to an existing table — you must
recreate and copy. `STRICT` still does not enforce length.

---

## Dates stored as strings

**Signal:** SQLite `created_at TEXT` holding `'2026-08-08 14:30'` with no offset. MongoDB documents
with `{ createdAt: "2026-08-08T14:30:00" }` as a string rather than `ISODate`.

**Why it breaks:** String dates only sort and range-filter correctly if every value uses an identical,
zero-padded, UTC ISO-8601 form. One `'2026-8-8'` or one local-time value silently breaks `BETWEEN`
and index ordering forever. In MongoDB a string date cannot be used with `$dateTrunc`, TTL indexes, or
any date operator, and mixed string/Date values in one field sort by BSON type before value.

**Fix:**

```sql
-- SQLite: one canonical UTC format, enforced
created_at TEXT NOT NULL
  CHECK (created_at IS strftime('%Y-%m-%dT%H:%M:%SZ', created_at))
```

```js
// MongoDB: BSON Date is a signed int64 of ms since epoch, always UTC
{ createdAt: new Date() }              // not new Date().toISOString()

db.c.updateMany({ createdAt: { $type: 'string' } },
  [{ $set: { createdAt: { $toDate: '$createdAt' } } }])
```

**Fine when:** the string is a full ISO-8601 UTC value with a trailing `Z`. Those sort
lexicographically exactly as they sort chronologically, and SQLite's date functions are built for
them. It is only a defect when the format is inconsistent, lacks an offset, or mixes local and UTC.
A MongoDB string is also acceptable for a date-only value like a birthday, where no zone applies.

**Confirm it (MongoDB, needs runtime):**

```js
db.coll.aggregate([{ $group: { _id: { $type: '$createdAt' }, n: { $sum: 1 } } }])
```

More than one type in the result is the finding.

---

## Booleans in MySQL and SQLite

**Signal:** `is_active BOOLEAN` or `TINYINT(1)` with no `CHECK`.

**Why it breaks:** MySQL has no boolean type — `BOOLEAN` and `BOOL` are aliases for `TINYINT(1)`,
which stores anything from -128 to 127. `UPDATE t SET is_active = 2` succeeds. Then
`WHERE is_active = TRUE` (that is, `= 1`) silently excludes those rows while `WHERE is_active`
(truthiness) includes them — two queries meant to be identical disagree. SQLite has the same lack of
enforcement.

**Fix:**

```sql
-- MySQL / MariaDB
is_active TINYINT(1) NOT NULL DEFAULT 0 CHECK (is_active IN (0,1))

-- SQLite
is_active INTEGER NOT NULL DEFAULT 0 CHECK (is_active IN (0,1))

-- Postgres: use the real type
is_active boolean NOT NULL DEFAULT false
```

**Fine when:** a single ORM is the only writer and its type layer already coerces to 0/1. The `CHECK`
is defence in depth, not a live bug.

Before MySQL 8.0.16, `CHECK` constraints were **parsed and ignored**, so a `CHECK` in an old migration
proves nothing about the running server. Decide the NULL policy explicitly too: a nullable boolean is
three-valued, and `WHERE is_active = false` will not return the NULL rows.

**Confirm it:** `SELECT DISTINCT is_active FROM t;` — anything other than 0, 1, and NULL is the bug.

---

## Postgres arrays as relationships

**Signal:** An array column modelling a relationship — `tag_ids integer[]`, `member_ids uuid[]`,
`role_ids bigint[]`.

**Why it breaks:** Postgres cannot put a foreign key on array elements, so deleting a tag leaves
dangling IDs in every row that referenced it, forever, with no error. Containment queries
(`tag_ids @> ARRAY[7]`) need a GIN index — a B-tree on an array column indexes the whole array as one
value and will never help — and updating one element rewrites the entire row.

**Fix:**

```sql
CREATE TABLE post_tags (
  post_id bigint REFERENCES posts(id) ON DELETE CASCADE,
  tag_id  bigint REFERENCES tags(id)  ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);
CREATE INDEX ON post_tags (tag_id);

-- if the array must stay, at minimum:
CREATE INDEX ON posts USING GIN (tag_ids);
```

**Fine when:** the list is small, ordered, self-contained, owned entirely by the row, and never joined
or referenced from elsewhere — `permissions text[]`, a weekday mask, an ordered `image_urls text[]`
where position is meaningful.

The disqualifying signal is specifically an `_ids` array pointing at another table's primary key, or
any query that needs to go from a tag back to its posts.

---

## MySQL `VARCHAR` length changes

**Signal:** `ALTER TABLE ... MODIFY col VARCHAR(256)` where the column was `VARCHAR(255)`, or any
MySQL `MODIFY`/`CHANGE` that alters a column's data type.

**Why it breaks:** InnoDB stores a VARCHAR's length in 1 byte when the maximum byte length is 0–255
and 2 bytes above that. Crossing that boundary changes the physical row format, so
`ALGORITHM=INPLACE` is refused and MySQL copies the whole table under a metadata lock — minutes to
hours of blocked writes from what looks like a one-character change. Decreasing a VARCHAR, and any
type change at all, is always `ALGORITHM=COPY`.

**Fix:** state the algorithm so the migration fails fast instead of blocking production.

```sql
ALTER TABLE t MODIFY c VARCHAR(200), ALGORITHM=INPLACE, LOCK=NONE;
```

If that errors, you know it is a rebuild — run it through `gh-ost` or `pt-online-schema-change`.

**Fine when:** the table is small or non-production, or you are widening within the same byte class
(`VARCHAR(100)` → `VARCHAR(200)` under latin1).

The boundary is in **bytes**, so under utf8mb4 it sits at 63 → 64 characters, not 255 → 256. In
Postgres, increasing a varchar's length limit has been a catalog-only change since 9.2, though
decreasing it still rewrites. SQLite has no VARCHAR length at all.

---

## Sources

PostgreSQL: [datatype-datetime](https://www.postgresql.org/docs/current/datatype-datetime.html),
[sql-altertype](https://www.postgresql.org/docs/current/sql-altertype.html),
[datatype-json](https://www.postgresql.org/docs/current/datatype-json.html),
[datatype-money](https://www.postgresql.org/docs/current/datatype-money.html).
MySQL: [datetime](https://dev.mysql.com/doc/refman/8.4/en/datetime.html),
[charset-unicode-utf8](https://dev.mysql.com/doc/refman/8.4/en/charset-unicode-utf8.html),
[innodb-limits](https://dev.mysql.com/doc/refman/8.4/en/innodb-limits.html),
[innodb-online-ddl-operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html).
SQLite: [datatype3](https://www.sqlite.org/datatype3.html),
[stricttables](https://www.sqlite.org/stricttables.html).
MongoDB: [bson-types](https://www.mongodb.com/docs/manual/reference/bson-types/).
