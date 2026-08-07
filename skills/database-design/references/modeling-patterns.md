# Modeling patterns

The recurring shapes, and the wrong version of each. Load when the schema has multi-tenancy, soft
deletes, money, hierarchies, status columns, audit trails, or translations.

This file is also the backbone of **design mode** — when modelling something new, find the pattern here
before inventing one.

Every entry has a **Fine when** line. Read it before reporting.

## Contents

- Soft deletes break unique constraints
- Multi-tenancy without composite foreign keys
- Postgres RLS that does nothing
- Schema-per-tenant
- Money
- Polymorphic associations
- EAV
- Status: enum vs varchar vs lookup table
- Multiple columns encoding one lifecycle
- Join tables
- Ordering and position columns
- Translations
- Hierarchies
- Under- and over-normalization
- Audit and history tables

---

## Soft deletes break unique constraints

**Signal:** a `deleted_at` / `is_deleted` column on a table that also has a plain `UNIQUE` on a business
key — `email TEXT UNIQUE, deleted_at TIMESTAMPTZ`.

**Why it breaks:** Once a row is soft-deleted the unique index still holds its value, so **the user can
never re-register that email** and the app returns a duplicate-key error for a row that, to them, no
longer exists. It surfaces weeks after launch, in production, on the sign-up path.

**Fix:**

```sql
-- Postgres / SQLite 3.8+
CREATE UNIQUE INDEX users_email_active ON users (email) WHERE deleted_at IS NULL;
```

```sql
-- MySQL has no partial indexes — use NULL-distinctness
ALTER TABLE users
  ADD COLUMN alive TINYINT GENERATED ALWAYS AS (IF(deleted_at IS NULL, 1, NULL)) STORED,
  ADD UNIQUE KEY (email, alive);
```

**Fine when:** the unique column is a surrogate id, already includes `deleted_at` in a composite key, or
the product genuinely never allows reuse of the value — some compliance rules require permanently
burning an identifier.

Prisma cannot express this natively without the `partialIndexes` preview feature; before that it must
live in raw migration SQL, and `prisma migrate diff` may keep trying to recreate it.

**Confirm it:** `SELECT email, count(*) FROM users GROUP BY 1 HAVING count(*) > 1;`

---

## Multi-tenancy without composite foreign keys

**Signal:** `tenant_id` on most tables, but foreign keys reference only `(id)` —
`project_id UUID REFERENCES projects(id)` on a table that also carries `tenant_id`.

**Why it breaks:** Nothing in the schema stops a row in tenant A pointing at a parent row in tenant B.
One missing `WHERE tenant_id = $1`, or one ID guessed in a URL, and data crosses tenants. The database
stores the corrupt link forever.

**Fix:**

```sql
ALTER TABLE projects ADD UNIQUE (tenant_id, id);
ALTER TABLE tasks ADD FOREIGN KEY (tenant_id, project_id) REFERENCES projects (tenant_id, id);

CREATE UNIQUE INDEX ON members (tenant_id, email);   -- scope business uniques per tenant
```

**Fine when:** the table is globally shared reference data (countries, currencies, plans) with no tenant
column, or the tenant is reachable only through one non-nullable parent **and** the app enforces scoping
in a single shared query layer.

Composite FKs make some ORMs — Prisma especially — awkward. Flag it, but note that a `CHECK` plus a
tested repository layer is a legitimate weaker alternative.

**Find it:** any single-column FK between two tenant-scoped tables.

---

## Postgres RLS that does nothing

**Signal:** `CREATE POLICY ... USING (tenant_id = current_setting('app.tenant')::uuid)` with
`ENABLE ROW LEVEL SECURITY` but **no `FORCE`**, while the app's `DATABASE_URL` user is the same role
that ran the migrations.

**Why it breaks:** **Table owners bypass RLS by default**, as do superusers and `BYPASSRLS` roles. The
policy looks correct in review and does absolutely nothing at runtime, because the app connects as the
table owner. This is the most common way an RLS-based tenancy design silently fails.

**Fix:**

```sql
ALTER TABLE tasks FORCE ROW LEVEL SECURITY;

CREATE ROLE app_user NOLOGIN NOBYPASSRLS;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

**Fine when:** the app already connects as a dedicated non-owner role. Supabase's `authenticated` and
`anon` roles are not table owners, so Supabase projects are usually safe here.

Referential-integrity checks — FK and unique validation — always bypass RLS by design, so a unique
constraint can still reveal that a row exists in another tenant. That is expected behaviour, not a bug.

**Confirm it:**

```sql
SELECT rolsuper, rolbypassrls FROM pg_roles WHERE rolname = current_user;
SELECT relname, relrowsecurity, relforcerowsecurity FROM pg_class
 WHERE relnamespace = 'public'::regnamespace;
```

> RLS *policy correctness* — who can read which rows — belongs to `backend-hardening`. This entry is
> only about the mechanism being inert.

---

## Schema-per-tenant

**Signal:** migration code interpolating a schema or database name — `CREATE SCHEMA tenant_${id}`,
`SET search_path TO ${tenant}` — or a connection factory building a URL per tenant.

**Why it breaks:** Every migration must run N times and can half-fail, leaving tenants on different
schema versions. Catalog bloat slows `pg_dump` and query planning past a few thousand schemas. And
per-tenant connections multiply the pool, which is fatal on serverless.

**Fix:** one schema, `tenant_id` everywhere.

```sql
CREATE TABLE tasks (
  id uuid PRIMARY KEY,
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  ...
);
CREATE INDEX ON tasks (tenant_id, created_at DESC);
```

**Fine when:** there are tens — not thousands — of tenants with contractual data-isolation or per-tenant
restore requirements, or one tenant's volume justifies separate storage. A hybrid is also legitimate:
shared tables for the long tail, a dedicated database for the two largest customers.

Do not flag `search_path` used for extensions (`SET search_path = public, extensions`).

---

## Money

**Signal:** a monetary column typed `FLOAT`, `DOUBLE PRECISION`, `REAL`, Prisma `Float`, or Postgres
`money`. Or a `numeric` amount with **no adjacent currency column**.

**Why it breaks:** Binary floats cannot represent 0.10 exactly, so sums drift and invoices fail to
reconcile by cents. Postgres `money` output is locale-sensitive via `lc_monetary`, so a dump restored
into a differently configured database can fail to load. A single-currency amount column becomes a data
migration the day a second currency appears.

**Fix:**

```sql
amount_minor BIGINT  NOT NULL,                    -- 1999 = $19.99
currency     CHAR(3) NOT NULL REFERENCES currencies(code),
CHECK (amount_minor >= 0)

-- or, if you need fractional-cent rates
unit_price NUMERIC(19,4) NOT NULL,
currency   CHAR(3)       NOT NULL
```

**Fine when:** the product is genuinely single-currency and that decision is documented.
`NUMERIC`/`DECIMAL` with an explicit scale is correct and needs no change — only floats and `money` are
defects.

**Watch the driver:** JavaScript `number` cannot hold a BIGINT of minor units above 2^53. With
`amount_minor BIGINT`, make sure the driver maps it to `bigint` or a string. Prisma maps `BigInt` to JS
`bigint`; node-postgres returns `int8` as a string by default.

---

## Polymorphic associations

**Signal:** a `*_type` + `*_id` column pair with no foreign key — `commentable_type VARCHAR,
commentable_id BIGINT`, Rails `belongs_to :commentable, polymorphic: true`, Laravel `morphTo()`.

**Why it breaks:** A polymorphic column **cannot have a foreign key**, because a FK targets exactly one
table. Nothing prevents `commentable_id = 99999` pointing at a row that never existed, so orphans
accumulate silently and joins require branching on the type string. Deleting the parent leaves the
children behind.

**Fix — an exclusive arc:** one nullable real FK per parent type.

```sql
CREATE TABLE comments (
  id       bigserial PRIMARY KEY,
  post_id  bigint REFERENCES posts(id)  ON DELETE CASCADE,
  photo_id bigint REFERENCES photos(id) ON DELETE CASCADE,
  body     text NOT NULL,
  CHECK (num_nonnulls(post_id, photo_id) = 1)
);
```

MySQL/SQLite: `CHECK ((post_id IS NOT NULL) + (photo_id IS NOT NULL) = 1)`.

**Fine when:** the parent types are genuinely many and open-ended — a generic audit log, activity feed,
or attachment store across 15+ tables. The exclusive arc becomes 15 nullable columns, which is worse.
There, keep the polymorphic pair but add `CHECK (commentable_type IN (...))` and a scheduled
orphan-sweep query.

Note that SQLite enforces no foreign key at all unless `PRAGMA foreign_keys = ON` runs on every
connection — so "we have FKs" may already be untrue there.

---

## EAV

**Signal:** a table whose columns are `entity_id`, `attribute`, and `value` — often with `value_text`,
`value_int`, `value_bool` siblings. Names like `custom_fields`, `properties`, `meta`.

**Why it breaks:** Every attribute lives in a different row, so the database cannot type-check,
`NOT NULL`, or foreign-key any of them, and reading one logical record needs one self-join per
attribute. Filtering on two attributes gets quadratically worse, and every value comes back as text for
the app to re-parse.

**Fix:**

```sql
-- real columns for anything the app knows about
ALTER TABLE products ADD COLUMN weight_grams int, ADD COLUMN colour text;

-- truly user-defined fields: one jsonb column
ALTER TABLE products ADD COLUMN custom jsonb NOT NULL DEFAULT '{}';
CREATE INDEX ON products USING gin (custom jsonb_path_ops);
```

**Fine when:** end users really do define their own fields at runtime — CRM custom fields, form builders
— though even then a single `jsonb` column beats EAV rows.

**Do not flag genuine key-value tables** that are not modelling entity attributes: feature flags, app
settings, i18n string catalogues, and session stores are correctly shaped this way.

**Signature of the anti-pattern:** a table with a `value` column, fewer than ~5 columns total,
referenced from many places.

---

## Status: enum vs varchar vs lookup table

**Signal:** `CREATE TYPE order_status AS ENUM (...)` for a status the product will keep extending. Or
the reverse — a bare `status VARCHAR(50)` with no `CHECK` and no lookup table.

**Why it breaks:** Postgres enums cannot have a value removed at all, and a value added inside a
transaction cannot be used until that transaction commits — which breaks migration tools that wrap each
migration in a transaction and then backfill. A bare varchar has the opposite problem: `'canceled'` vs
`'cancelled'` are both stored happily.

**Fix:**

```sql
status text NOT NULL CHECK (status IN ('draft','paid','shipped','cancelled'))

-- or, when statuses carry metadata (labels, ordering)
status text NOT NULL REFERENCES order_statuses(code)
```

Adding a value later is then one fully transactional `ALTER`.

**Fine when:** the set is closed and stable and will never shrink — `'asc'/'desc'`, weekday names. Those
cost 4 bytes instead of a string. Do not flag them.

If you do flag one, say that migrating an existing enum column to text is **itself** a rewrite:
`ALTER TABLE ... ALTER COLUMN status TYPE text` takes `ACCESS EXCLUSIVE` and rewrites the table.

---

## Multiple columns encoding one lifecycle

**Signal:** `status` alongside `is_paid`, `is_shipped`, `cancelled_at`, `completed BOOLEAN`.

**Why it breaks:** The columns drift out of agreement — rows end up `status='cancelled'` with
`is_paid = true` — and no query can then answer "which orders are actually live." Every reader guesses
which column is authoritative, and different parts of the app guess differently.

**Fix:** one authoritative column. Timestamps are evidence, not state.

```sql
status text NOT NULL CHECK (status IN ('draft','paid','shipped','cancelled')),
paid_at      timestamptz,
cancelled_at timestamptz,
CHECK ((status = 'paid')      = (paid_at IS NOT NULL)),
CHECK ((status = 'cancelled') = (cancelled_at IS NOT NULL))

-- "only one active subscription per user"
CREATE UNIQUE INDEX ON subs (user_id) WHERE status = 'active';
```

**Fine when:** the booleans are genuinely independent dimensions — `is_verified` and `is_admin` are not
the same state machine as `status`. Timestamp columns beside a status column are good practice; the
finding is only when a boolean **duplicates** information the status column already carries.

Enforcing legal transitions in the database is usually overkill for this audience. One status column
plus consistency `CHECK`s is the right depth.

---

## Join tables

**Signal:** a join table with a surrogate `id` primary key and no unique constraint on the pair. Or a
Prisma implicit many-to-many (`tags Tag[]` on both sides, generating `_PostToTag`).

**Why it breaks:** Without a unique on `(post_id, tag_id)` the same tag can be attached twice, and
"remove tag" then deletes one of two rows. Prisma's implicit relation table **cannot hold extra columns
at all**, so the day you need `added_at` you must convert it to an explicit model and migrate the
generated table by hand.

**Fix:**

```sql
CREATE TABLE post_tags (
  post_id  int NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id   int NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
  added_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (post_id, tag_id)
);
CREATE INDEX ON post_tags (tag_id);     -- reverse lookups
```

**Fine when:** another table needs to foreign-key the association itself, or the ORM requires a
single-column primary key. Keep the `id` but add `UNIQUE (post_id, tag_id)` — **that** is the actual fix.

Do not skip the second index: a composite primary key only serves lookups leading with `post_id`.

---

## Ordering and position columns

**Signal:** an integer `position` / `sort_order` / `rank` column with no unique constraint, plus code
doing `UPDATE items SET position = position + 1 WHERE position >= ?` on every insert.

**Why it breaks:** The renumber rewrites every following row on each insert, and two concurrent reorders
interleave into duplicate or skipped positions, so the list renders differently for different users.
With a naive unique constraint, swapping A and B fails mid-statement because the intermediate state
collides.

**Fix:**

```sql
-- fractional keys: insert between neighbours, no rewrite
position numeric NOT NULL,          -- 1.0, 2.0, 1.5, 1.25
UNIQUE (parent_id, position)

-- or keep integers and defer the constraint (Postgres only)
ALTER TABLE items ADD CONSTRAINT items_pos_uq
  UNIQUE (parent_id, position) DEFERRABLE INITIALLY DEFERRED;
```

A unique **index** cannot be deferred — only a unique **constraint**. MySQL and SQLite have no
deferrable unique at all; use fractional keys there.

**Fine when:** ordering is derived rather than stored (`ORDER BY created_at DESC`), or the list is short
and rarely reordered — renumbering 20 menu items costs nothing.

Fractional positions drift toward long decimals after many insertions at the same spot. A periodic
renormalisation job, or a lexicographic key (LexoRank-style), handles that.

---

## Translations

**Signal:** per-language columns — `title_en`, `title_fr` — or a single `jsonb` column of locale keys
with no constraint on which locales it may contain.

**Why it breaks:** Adding a language becomes a schema migration touching every table, and every query
must know the locale list at compile time. Sorting also breaks: `ORDER BY title` uses the database's
default collation, which is wrong for languages with accents or non-Latin scripts.

**Fix:**

```sql
CREATE TABLE product_translations (
  product_id  int  NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  locale      text NOT NULL,            -- BCP-47: 'en', 'pt-BR'
  name        text NOT NULL,
  description text,
  PRIMARY KEY (product_id, locale)
);

ORDER BY name COLLATE "de-DE-x-icu";    -- locale-correct sorting, Postgres 10+
```

**Fine when:** the product is contractually bilingual and will never add a third language — flag that as
Low, not a defect. A `jsonb` translations column is also fine for content only ever read whole by the
app and never filtered or sorted in SQL.

The finding is real when the app filters or sorts on translated text, or adding a locale is on the
roadmap.

---

## Hierarchies

**Signal:** a self-referencing `parent_id` with recursive queries and no depth or cycle guard — look for
`WITH RECURSIVE` with no `CYCLE` clause and no depth counter.

**Why it breaks:** A row that becomes its own ancestor — easy to create with a drag-and-drop "move
folder" feature — makes the recursive query loop until it exhausts memory and takes the server down.
Deep adjacency-list trees also cost one recursion level per depth, which is why "load the sidebar" gets
slow at ~6 levels.

**Fix:**

```sql
-- Postgres 14+ has built-in cycle detection
WITH RECURSIVE t AS (
  SELECT id, parent_id FROM nodes WHERE id = $1
  UNION ALL
  SELECT n.id, n.parent_id FROM nodes n JOIN t ON n.parent_id = t.id
) CYCLE id SET is_cycle USING path
SELECT * FROM t WHERE NOT is_cycle;

-- or ltree for read-heavy trees
path ltree NOT NULL;
CREATE INDEX ON nodes USING gist (path);
```

MySQL 8.0 and SQLite 3.8.3+ have `WITH RECURSIVE` but **no `CYCLE` clause** — use an explicit depth cap
there.

**Fine when:** the tree is shallow and read once per page. **Adjacency list is the right default** — do
not recommend a closure table for a 3-level category tree.

Escalate only when the tree is deep, reordered by users, or queried for whole subtrees on a hot path.
State the tradeoff plainly: closure tables make subtree reads O(1) joins but every move rewrites
O(descendants × ancestors) rows; ltree makes reads fast and moves cheap but needs a trigger to keep
descendant paths in sync.

`ltree` needs `CREATE EXTENSION` and a **GiST** index — a B-tree does not serve `@>`, `<@`, or `~`.

---

## Under- and over-normalization

**Under-normalized signals:** a comma-separated list in a text column (`tags VARCHAR(255)` holding
`'red,blue'`), numbered columns (`phone1, phone2, phone3`), or a status *name* repeated as free text on
every row.

**Over-normalized signals:** a 1:1 side table with a primary key and one other column, joined in
literally every query. A lookup table of two rows.

**Why it breaks:** Comma lists cannot be indexed, foreign-keyed, or queried without `LIKE '%red%'` —
which also matches "redwood". Numbered columns cap the count arbitrarily and force UNIONs. The opposite
failure is cheaper but real: a 1:1 split with identical cardinality just adds a join to every read and a
second write path, with no integrity gain.

**Fix:**

```sql
-- under-normalized: promote to rows
CREATE TABLE post_tags (
  post_id int REFERENCES posts(id) ON DELETE CASCADE,
  tag_id  int REFERENCES tags(id),
  PRIMARY KEY (post_id, tag_id));

-- over-normalized 1:1 with no cardinality difference: fold the child back into the parent
```

**Fine when:** the 1:1 side table holds columns that are large (blobs, long text) and rarely read,
optional for most rows, or need different access control. A `text[]`/`jsonb` array of tags is also
correct if tags never need their own rows — no rename, no description, no per-tag permissions.

**The distinguishing question is cardinality and independent lifetime, not table count.**

---

## Audit and history tables

**Signal:** an `audit_log` / `history` table written from application code rather than a trigger, or
history rows carrying a real `FOREIGN KEY` back to the live table.

**Why it breaks:** Application-written audit rows are missing for every path that does not go through
the ORM — raw SQL, admin console, migrations, background jobs — so the trail is **trusted and
incomplete**. A FK from history to the live table makes it impossible to delete the subject row without
either failing or cascading away its own history.

**Fix:**

```sql
CREATE TABLE orders_history (LIKE orders);
ALTER TABLE orders_history
  ADD COLUMN valid_from timestamptz NOT NULL DEFAULT now(),
  ADD COLUMN valid_to   timestamptz,
  ADD COLUMN changed_by text;
-- populate from an AFTER INSERT OR UPDATE OR DELETE trigger
-- no FK back to the live table, no ON DELETE
```

**Fine when:** the app is the only writer and the audit exists for a product feature (an activity feed)
rather than compliance. **Say which one it is.**

Triggers have a real cost — they roughly double write volume and hide behaviour from developers — so do
not recommend them for high-write tables without saying so. And `LIKE orders` copies the column list at
creation time only: the history table drifts when the live table gains a column unless the trigger is
regenerated.

MariaDB 10.3+ has `WITH SYSTEM VERSIONING` and SQL Server 2016+ has temporal tables. Postgres gained
`WITHOUT OVERLAPS` temporal constraints in 18; MySQL and SQLite have neither.

---

## Sources

PostgreSQL: [sql-createtable](https://www.postgresql.org/docs/current/sql-createtable.html),
[ddl-rowsecurity](https://www.postgresql.org/docs/current/ddl-rowsecurity.html),
[datatype-money](https://www.postgresql.org/docs/current/datatype-money.html),
[ltree](https://www.postgresql.org/docs/current/ltree.html),
[queries-with](https://www.postgresql.org/docs/current/queries-with.html),
[18.0 release notes](https://www.postgresql.org/docs/release/18.0/).
MySQL: [create-index](https://dev.mysql.com/doc/refman/8.4/en/create-index.html),
[generated-columns](https://dev.mysql.com/doc/refman/8.4/en/create-table-generated-columns.html).
SQLite: [partialindex](https://www.sqlite.org/partialindex.html),
[foreignkeys](https://www.sqlite.org/foreignkeys.html).
Prisma: [indexes](https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes).
