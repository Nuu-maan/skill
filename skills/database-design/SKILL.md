---
name: database-design
description: Reviews a database for schema, indexing, query performance, and migration-safety problems, explains each one in plain language, then fixes them on request — and designs a new schema from scratch when there is nothing to review yet. Covers PostgreSQL, MySQL/MariaDB, SQLite, MongoDB, and Supabase, plus Prisma, Drizzle, TypeORM, Mongoose, Django ORM, SQLAlchemy, ActiveRecord, Hibernate, and EF Core. Use whenever the user asks to review, audit, or improve a database or schema; asks why a query, page, or endpoint is slow; mentions missing indexes, N+1 queries, table scans, or a migration that locked a table; asks how to model or structure data for a new feature; asks whether their schema is right before they have real users; or has an AI-generated schema they do not fully understand. Also use proactively when writing or editing migrations, schema files, or ORM model definitions.
license: MIT
compatibility: Requires filesystem read access and a code-search tool such as grep or ripgrep. Never connects to a database — findings that need runtime data are produced as SQL for the user to run and paste back.
allowed-tools: Read, Grep, Glob
metadata:
  version: 1.0.0
  author: nuu-maan
  homepage: https://qala.lol
---

# Database Design

Review a database and report what is actually wrong, or design a new one. Fix only with approval.

**Assume the user is strong at frontend and weak at databases.** They likely generated this schema
with an AI and cannot judge whether a finding is real. That has three consequences:

1. **A false positive is worse than a miss.** They cannot filter your output. Never report a finding
   you have not confirmed against the actual schema and the actual query.
2. **Every finding needs a plain-language "so what."** State the concrete outcome — "totals drift by
   a cent per order and your payouts will not reconcile" — not the category name.
3. **Never edit schema or migrations without asking.** A bad migration is the one mistake in this
   domain that destroys data rather than exposing it.

## Operating rules

- Every finding cites `file:line` and quotes the offending definition. No finding without evidence.
- **You do not have a database connection.** Everything comes from files. When a finding depends on
  runtime facts — real row counts, actual query plans, which indexes exist in production — write the
  exact SQL for the user to run and say what answer would confirm it. Never assume table size.
- Severity is about data loss, wrong results, and scale — not tidiness. See the rubric below.
- Never invent engine behavior. Engines differ sharply on defaults; if you are not certain a version
  behaves a given way, say so rather than asserting it.
- Never propose a migration without saying whether it locks the table and for how long.
- Do not redesign a working schema because you prefer a different shape. Fix defects.

## Pick the mode

| Mode | When | Go to |
| :--- | :--- | :--- |
| **Review** | A schema, migrations, or ORM models already exist | [Review workflow](#review-workflow) |
| **Design** | Modeling something new, or no database exists yet | [Design workflow](#design-workflow) |

Doing both is normal: review the existing schema, then design the new table it needs. Run Review
first so the design fits what is already there.

---

## Review workflow

Copy this checklist into your reply and tick items as you go:

```
Database review:
- [ ] R1: Map the database
- [ ] R2: Fast sweep (highest-signal checks)
- [ ] R3: Deep audit (load only relevant reference files)
- [ ] R4: Verify every candidate finding
- [ ] R5: Report, ranked by severity
- [ ] R6: Offer to fix, then fix in priority order
```

### R1: Map the database

Identify these from files, not from guessing:

| What | Where to look |
| :--- | :--- |
| Engine + version | `docker-compose.yml`, `*.tf`, connection strings, `package.json`/`requirements.txt` driver, CI service images |
| Schema definition | `migrations/`, `schema.sql`, `schema.prisma`, `schema.rb`, `models/`, `entities/`, `*.dbml` |
| ORM / query layer | Dependency manifest, plus raw SQL strings anywhere in the codebase |
| Access patterns | The queries the app actually runs — route handlers, repositories, serializers |
| Scale signals | Seed/fixture size, pagination defaults, any comment about row counts, analytics tables |

Then load **one engine file and one ORM file**:

| Engine | File |
| :--- | :--- |
| PostgreSQL, Supabase, Neon, RDS Postgres | [engines/postgres.md](engines/postgres.md) |
| MySQL, MariaDB, PlanetScale | [engines/mysql.md](engines/mysql.md) |
| SQLite, Turso, libSQL, D1 | [engines/sqlite.md](engines/sqlite.md) |
| MongoDB, Atlas, DocumentDB | [engines/mongodb.md](engines/mongodb.md) |

| ORM / query layer | File |
| :--- | :--- |
| Prisma, Drizzle, TypeORM, Sequelize, Mongoose, Kysely | [orms/typescript.md](orms/typescript.md) |
| Django ORM, SQLAlchemy, Alembic | [orms/python.md](orms/python.md) |
| ActiveRecord, Hibernate/JPA, EF Core | [orms/ruby-jvm-dotnet.md](orms/ruby-jvm-dotnet.md) |

Raw SQL with no ORM: skip the ORM file.

### R2: Fast sweep

These twelve checks catch most real defects in AI-generated schemas. Run them on every review,
regardless of engine.

1. **Money in a float.** `FLOAT`, `DOUBLE`, `REAL`, Prisma `Float`, Mongoose `Number` on a column
   named `price`, `amount`, `total`, `balance`, `fee`, `cost`. Sums drift; equality comparisons miss.
2. **Timestamps without a zone.** Postgres `timestamp` instead of `timestamptz`, MySQL `TIMESTAMP`
   on a column that must hold dates past 2038, naive `datetime` in application code.
3. **Foreign key columns with no index.** Postgres does **not** create one for you. Check every
   `REFERENCES` / relation against the index list.
4. **Everything nullable.** Columns the application always sets, declared without `NOT NULL`.
5. **Missing unique constraints.** Uniqueness the code assumes (`email`, `slug`, `(tenant_id, name)`)
   enforced only in application code, or not at all.
6. **No foreign keys at all.** Especially SQLite, where `PRAGMA foreign_keys` is **off by default**
   and declared keys are silently ignored.
7. **Free-text status columns.** A `status` or `role` column as unconstrained text, with no `CHECK`,
   enum, or lookup table.
8. **N+1 queries.** A query inside a loop, or a relation accessed per item in a serializer.
9. **Unbounded reads.** `findAll()`, `.objects.all()`, `SELECT *` with no `LIMIT`, an endpoint that
   returns every row, or `OFFSET` pagination on a table that will grow.
10. **Dangerous migrations.** `CREATE INDEX` without `CONCURRENTLY` on Postgres, `ADD COLUMN NOT NULL`
    on a large table, `synchronize: true`, `ddl-auto: update`, or a migration edited after it ran.
11. **Soft deletes applied inconsistently.** A `deleted_at` column that some queries filter on and
    some do not — and unique constraints that ignore it.
12. **Schema-avoidance.** A JSON/JSONB column holding fields that are queried, filtered, or joined on,
    or a MongoDB collection with no `$jsonSchema` validation and no application-level schema.

Anything the sweep hits, confirm in R4 before it becomes a finding.

### R3: Deep audit

Load **only the reference files matching what this codebase actually does.** Each is a checklist of
signal → why it breaks → fix. They cost nothing until read.

| Load this | When the codebase has |
| :--- | :--- |
| [references/schema-and-types.md](references/schema-and-types.md) | Any column definition — types, precision, charsets, JSON, arrays |
| [references/constraints-and-keys.md](references/constraints-and-keys.md) | Primary keys, foreign keys, uniqueness, defaults, generated columns |
| [references/modeling-patterns.md](references/modeling-patterns.md) | Multi-tenancy, soft deletes, money, hierarchies, audit trails, status machines |
| [references/indexes.md](references/indexes.md) | Any index, or any query filtering, joining, or sorting |
| [references/query-performance.md](references/query-performance.md) | Slowness, pagination, aggregates, search, or any ORM query code |
| [references/reading-plans.md](references/reading-plans.md) | The user pasted a query plan, or you asked them to run `EXPLAIN` |
| [references/migrations.md](references/migrations.md) | A migrations folder, or any proposed schema change |

### R4: Verify

For each candidate finding, before it goes in the report:

1. Read the whole table definition and every migration that touched it — not just the matched line.
   A later migration may already have added the index or constraint.
2. Confirm the column is actually used the way you assume. An unindexed column nothing filters on is
   not a finding.
3. Check the engine and version. Most defaults in this domain are version-specific.
4. Check the `caveat` for the pattern in the reference file — most have a legitimate use.
5. If it depends on data volume, say so and give the SQL to check. Do not guess at row counts.
6. If you still cannot prove it, drop it or label it "unconfirmed — needs you to check X."

Then rate severity:

| Severity | Definition |
| :--- | :--- |
| **Critical** | Data is being lost, corrupted, or silently wrong right now, or the next migration will take production down. Fix before deploying. |
| **High** | Wrong results under normal use — duplicate rows, drifting money, orphaned records — or a query that collapses at realistic scale. |
| **Medium** | Degrades as the table grows, or makes a future change painful. Nothing breaks today. |
| **Low** | Hygiene and consistency. Real, but no user notices. |

A missing index on a 200-row lookup table is Low. Money in a float is Critical. Do not inflate.

### R5: Report

Use the exact structure in [templates/review.md](templates/review.md). Lead with a verdict and a
severity count so the user knows immediately whether they can ship.

### R6: Offer to fix

End with a concrete offer, not an open question:

> I can fix these. Suggested order: the money column first (one migration, needs a backfill), then
> the three missing indexes (safe, no downtime). Want me to start?

Then follow [references/fixing-safely.md](references/fixing-safely.md).

---

## Design workflow

Use when modeling something new. Copy this checklist into your reply:

```
Schema design:
- [ ] D1: Extract the entities
- [ ] D2: Define each lifecycle (states, and what "deleted" means)
- [ ] D3: Write the access patterns — before any DDL
- [ ] D4: Fix cardinality, ownership, and fan-out
- [ ] D5: Choose keys
- [ ] D6: Turn every rule into a constraint
- [ ] D7: Derive indexes from D3
- [ ] D8: Validate, then deliver
```

The full procedure — each step's worked example, and the specific mistake made at each — is in
[references/designing-a-schema.md](references/designing-a-schema.md). **Load it before starting.**

**The rule that matters most:** finish D3 before writing any schema. A schema is a bet on the
questions you will ask. In MongoDB this is decisive — the document shape *is* the access pattern. In
SQL it decides your indexes and saves the migration you would otherwise write in three months.

D1–D3 are the steps that cannot be recovered later. D5–D7 are mechanical once D1–D3 exist.

Ask the user for what you cannot infer: expected scale, who owns what, what must never be duplicated,
what has to stay correct under concurrency. Do not invent these.

Deliver using [templates/design.md](templates/design.md).

---

## Not this skill

- **Security** — who can read which rows, RLS policies, SQL injection, exposed credentials. That is
  `backend-hardening`. If the request is "is my database safe", use that skill instead.
- **Transaction correctness and race conditions** at runtime — lost updates, missing transactions,
  idempotency. Also `backend-hardening/references/data-layer.md`.
- **Query tuning against a live database.** This skill reads files. It can interpret a plan the user
  pastes in, but it cannot run one.

## When there is nothing to find

Say so plainly and show your work — which tables you read, which access patterns you traced, what you
could not verify without row counts. "The schema is sound; three things will matter at 100k rows" is
a useful result. Do not manufacture findings to fill the report.
