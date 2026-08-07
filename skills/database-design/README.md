# database-design

Reviews a database for schema, indexing, query, and migration-safety problems, explains each one in
plain language, then fixes them on request — and designs a new schema from scratch when there is nothing
to review yet.

Built for developers who are strong at frontend and learning backend, including people whose schema was
largely written by an AI. It reports before it edits, every finding comes with a concrete consequence,
and every migration it proposes says whether it locks the table.

## Two modes

| Mode | Trigger | What you get |
| :--- | :--- | :--- |
| **Review** | "check my schema", "why is this slow", "is this ready to launch" | A severity-ranked report with `file:line` evidence, a fix per finding, and a self-check command |
| **Design** | "how should I model X", "I need a schema for Y" | Access patterns first, then schema, then indexes justified by those patterns, then a walkthrough proving it answers each one |

Doing both is normal — review what exists, then design the table it needs.

## What it covers

| Area | Examples |
| :--- | :--- |
| Types | Money in floats, `timestamp` vs `timestamptz`, MySQL's 2038 limit, `utf8` vs `utf8mb4`, UUID storage and index locality, SQLite type affinity |
| Constraints | Foreign keys SQLite silently ignores, `CHECK` MySQL 5.7 ignores, `UNIQUE` on nullable columns, generated columns, primary key choice |
| Modeling | Multi-tenancy, soft deletes vs unique constraints, polymorphic associations, EAV, status columns, hierarchies, translations, audit tables |
| Indexes | Unindexed foreign keys, composite column order, MongoDB's ESR rule, partial and expression indexes, redundant and unused indexes |
| Queries | `NOT IN` with NULLs, OFFSET pagination, `count(*)`, N+1, `DISTINCT` over a fan-out join, unanchored `LIKE`, huge `IN` lists |
| Query plans | Reading `EXPLAIN` in all four engines, with a numeric threshold for each signal |
| Migrations | What locks and for how long, per engine and version; expand-and-contract; batched backfills; editing an applied migration |

Engine-specific files for **PostgreSQL** (and Supabase), **MySQL/MariaDB** (and PlanetScale),
**SQLite** (and Turso/D1), and **MongoDB**. ORM-specific files for **Prisma, Drizzle, TypeORM,
Sequelize, Mongoose, Kysely**; **Django ORM, SQLAlchemy, Alembic**; **ActiveRecord, Hibernate/JPA,
EF Core**.

## Install

```bash
npx skills add https://qala.lol/numan/database-design
```

Works in any agent that supports the [Agent Skills standard](https://agentskills.io) — Claude Code,
Cursor, Codex, Copilot, Gemini CLI, OpenCode, Goose, and others.

Or, in Claude Code, install the whole collection as a plugin:

```
/plugin marketplace add Nuu-maan/skill
/plugin install qala@qala
```

## Use

Ask in your own words — the skill triggers on requests like:

- "Review my database schema"
- "Why is this endpoint slow?"
- "Is my schema right before I get users?"
- "How should I model comments on posts?"
- "Will this migration take the site down?"

Or invoke it directly in Claude Code:

```
/database-design
```

You can scope it: `/database-design just the indexes` or
`/database-design design a schema for a booking system`.

## What to expect

**Reviewing:** it identifies the engine and ORM from your files, runs twelve high-signal checks, loads
only the deep-dive files that match what your code actually does, verifies each candidate against the
real definition, then reports in severity order and offers to fix.

**Designing:** it asks what it cannot infer — scale, ownership, what must never be duplicated, what has
to stay correct under concurrency — then works through entities, lifecycles, access patterns,
relationships, keys, constraints, and indexes in that order, and walks each access pattern through the
result before handing it over.

If it finds nothing serious it says so, rather than padding the report.

## It never connects to your database

Everything comes from files: schema definitions, migrations, ORM models, and query code. When a finding
depends on runtime facts — real row counts, actual query plans, which indexes exist in production — it
writes the exact SQL for you to run and tells you what answer would confirm it.

That is a deliberate limit, not an oversight. It means the skill needs no credentials and can never
touch your data.

## Claude Code variant

The canonical `SKILL.md` uses only the six fields in the Agent Skills spec, so it works everywhere. If
you use it exclusively in Claude Code, these frontmatter additions are worth considering:

```yaml
# Run the review in an isolated subagent so the (long) reading phase does not
# consume your main conversation's context.
context: fork
agent: Explore
background: false
```

```yaml
# Load automatically only when working in schema or migration files.
paths: ["**/migrations/**", "**/*.sql", "**/schema.prisma", "**/models/**", "**/entities/**"]
```

Do **not** add these to a copy you upload to claude.ai or the Skills API — those paths reject non-spec
keys with a hard error.

## Boundaries

This skill does **not** cover:

- **Security** — who can read which rows, RLS policy correctness, SQL injection, exposed credentials.
  That is [backend-hardening](../backend-hardening).
- **Runtime transaction correctness** — missing transactions, lost updates, idempotency. Also
  backend-hardening.
- **Live query tuning.** It can interpret a plan you paste in; it cannot run one.

It is also not a substitute for load testing. It removes the errors that are cheap to find early.

## Files

```
database-design/
├── SKILL.md                        # entry point: mode router, both workflows, severity rubric
├── references/                     # loaded only when relevant
│   ├── schema-and-types.md
│   ├── constraints-and-keys.md
│   ├── modeling-patterns.md
│   ├── indexes.md
│   ├── query-performance.md
│   ├── reading-plans.md
│   ├── migrations.md
│   ├── designing-a-schema.md
│   └── fixing-safely.md
├── engines/                        # one is loaded, matching the project
│   ├── postgres.md
│   ├── mysql.md
│   ├── sqlite.md
│   └── mongodb.md
├── orms/                           # one is loaded, matching the project
│   ├── typescript.md
│   ├── python.md
│   └── ruby-jvm-dotnet.md
└── templates/
    ├── review.md
    └── design.md
```

Only `SKILL.md` enters context when the skill triggers. Everything else is read on demand, so the depth
costs nothing on a small review.

Every entry in every reference file carries a **Fine when** line stating exactly when the pattern is
legitimate. That is deliberate: the target reader cannot filter a false positive, so a wrong finding
costs them more than a missed one.

## License

MIT
