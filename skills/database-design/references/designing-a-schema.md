# Designing a schema

The ordered procedure for turning a plain-language feature description into a schema. Load before
starting design mode.

**The order is the method.** Each step produces the input to a later one. The most common way an
AI-written schema goes wrong is jumping straight to `CREATE TABLE` — which means entities, lifecycles,
and access patterns were all decided implicitly and none of them were written down.

Steps 1–3 are the ones that cannot be recovered later. Steps 5–8 are mechanical once 1–3 exist.

## Contents

1. Extract the entities
2. Define each lifecycle
3. Write the access patterns — **before any DDL**
4. Fix cardinality, ownership, and fan-out
5. Choose keys
6. Turn every rule into a constraint
7. Derive indexes from step 3
8. Validate, then deliver

---

## Step 1 — Extract the entities

**Do this:** rewrite the feature description as sentences of the form
`<actor> <verb> <thing> [with/for <thing>]`.

- A noun that is the object of **more than one** action, or that anything else points at, is an
  **entity**.
- A noun that only ever appears as a property of one other noun is an **attribute**, not a table.

**Worked example:**

```
Prose:    "A user can create a project, invite teammates to it,
           and post comments on tasks inside it."

Actions:  user creates project | user invites user to project
          user posts comment on task | task belongs to project

Entities:   user, project, membership, task, comment
Attributes: project.name, comment.body       (not tables)
```

**Why it is unrecoverable:** an attribute promoted to a table adds a join to every query forever. An
entity demoted to a JSON blob cannot be queried, constrained, or indexed independently.

**The mistake made here:** over-extraction. A table with exactly one foreign key, nothing referencing
it, and fewer than three non-key columns is probably an attribute that got its own table.

**Legitimate exceptions:** lookup tables that must be referenced by a foreign key; 1:1 splits isolating
large blobs or PII; a genuinely schemaless payload (a webhook body kept verbatim) as one JSON column.
The test for that last one is whether the application ever filters or joins on its contents.

---

## Step 2 — Define each lifecycle

**Do this:** for every entity, write down how it is created, what states it passes through, who may
change it, and **what "deleted" means** — before writing any columns.

**Worked example:**

```
order: draft -> submitted -> paid -> refunded (terminal)
Never hard-deleted (financial record) -> no DELETE, use 'cancelled'
```

```sql
status       text not null default 'draft'
  check (status in ('draft','submitted','paid','refunded','cancelled')),
created_at   timestamptz not null default now(),
submitted_at timestamptz            -- null until that transition happens
```

**Why it matters:** this produces the status column, the timestamps that actually matter, and the delete
policy. Skipping it is the single most common cause of a schema being rewritten later, because
hard-delete vs soft-delete vs archive changes **every uniqueness constraint, every `ON DELETE` clause,
and every query's `WHERE`**.

**The mistake made here:** assuming soft delete. It is not always right — for GDPR erasure, join tables,
and high-churn ephemeral rows (sessions, rate-limit counters, outbox rows), hard delete is correct and
simpler. And an entity with exactly one state needs no status column at all.

**Two checks:** every timestamp column should correspond to a named transition — orphan timestamps mean
the lifecycle was never written down. And a `deleted_at` column requires a partial unique index
alongside it (see [modeling-patterns.md](modeling-patterns.md)).

---

## Step 3 — Write the access patterns

**Do this before writing any DDL.** One row per operation the application will actually issue.

| Action | R/W | Filter | Sort | Freq/day |
| :--- | :--- | :--- | :--- | ---: |
| Open project board | R | `project_id`, `status <> 'done'` | `position` | 500k |
| Task detail + comments | R | `task_id` | `created_at` | 200k |
| Mark task done | W | `task_id` | — | 50k |
| Admin: overdue report | R | `due_at < now` | `due_at` | 20 |

**Why this is the pivot of the whole procedure:** it is the input to the index step, to the
embed-vs-reference decision, and to validation. Without it every later decision is a guess.

MongoDB's own schema design process makes workload identification **step 1 of 4**, because document
shape is chosen to serve queries and cannot be joined around later. In SQL the same list is merely
important — a wrong shape can usually be recovered with a join or an added index.

**If the code already exists,** reconstruct the list rather than asking: each distinct filter + sort
combination is one access pattern. If the repo is new, **the API route list is the access-pattern
list.**

**The mistake made here:** trying to enumerate everything. Frequency estimates only need to be right to
an order of magnitude. Patterns running under ~100 times a day should not influence schema shape or earn
an index. This is a starting hypothesis, not a substitute for measuring later.

---

## Step 4 — Fix cardinality, ownership, and fan-out

**Do this:** for every pair of related entities, decide 1:1 / 1:N / M:N, whether the child can exist
without the parent, what happens to the child when the parent goes away, and **how large N can get**.

```
project 1--N task      (task.project_id NOT NULL, owned)
task    1--N comment   (comment.task_id NOT NULL, owned)
user    1--N comment   (comment.author_id NULL, not owned)
```

```sql
project_id uuid not null references project(id) on delete cascade,
author_id  uuid     null references "user"(id) on delete set null
```

Cardinality decides **where the foreign key lives** — always on the N side — and whether a join table is
needed. Ownership decides `ON DELETE` and nullability.

**Then estimate fan-out and growth:**

```
user      ~100k in y1, +10k/mo
project   ~5 per user, bounded
task      ~500 per project, growing
comment   unbounded per task        -> reference, never embed
event_log ~2M/mo, unbounded         -> own table + retention job
```

**Why fan-out matters:** it decides embed vs reference in MongoDB and whether a relationship needs its
own index or pagination in SQL. Unbounded arrays are the classic MongoDB failure — documents grow until
they hit the 16 MB BSON limit and updates get progressively slower.

**The mistake made here:** leaving `ON DELETE` unstated. The default is `NO ACTION`/`RESTRICT`, which is
a choice nobody made. Flag every foreign key with no explicit `ON DELETE`, and every nullable FK that
does not correspond to a real "this may be unattached" case from step 2.

`ON DELETE RESTRICT` is often right even for owned data — it turns an accidental parent delete into a
loud error instead of silent data loss. `SET NULL` is only valid on a nullable FK.

**Do not over-engineer on the estimates.** Partitioning, sharding, and archival tiers are almost always
premature below roughly 10–50M rows. The estimate's job here is only to catch unbounded cases.

---

## Step 5 — Choose keys

**Do this:** pick a surrogate primary key. Give natural business identifiers their own `UNIQUE`
constraint. Decide these separately.

```sql
-- Postgres 18+: time-ordered UUID, safe to expose
id uuid primary key default uuidv7()

-- earlier, or a single-database app
id bigint generated always as identity primary key

-- the natural key is a constraint, not the PK
email citext not null unique

-- join table: composite PK, no surrogate id
primary key (project_id, user_id)
```

**Why:** the primary key is what every foreign key, index, and cache key depends on, so it must never
change — and business identifiers do change. People change emails; slugs get renamed.

Key type also determines index locality. Random UUIDv4 scatters B-tree inserts and bloats indexes;
bigint identity and UUIDv7 append at the end. **In MySQL/InnoDB the effect is stronger than Postgres**,
because the primary key is the clustered index — random keys scatter the table itself, not just an
index.

**The mistake made here:** a natural column (`email`, `slug`, `code`) declared as `PRIMARY KEY`.

**Fine when:** UUIDv4 at low write volume, or when IDs are generated client-side or merged across
databases. In MongoDB the default `ObjectId` is already time-ordered.

---

## Step 6 — Turn every rule into a constraint

**Do this:** go through the feature description sentence by sentence and ask, for each rule, *which
constraint enforces this?*

```sql
-- "a project name is required and unique per owner"
name text not null check (length(trim(name)) > 0),
unique (owner_id, name)

-- "...but only among non-deleted projects"
create unique index on project (owner_id, name) where deleted_at is null;

-- "quantity must be positive"
check (quantity > 0)
```

**Why:** constraints are the only rules that hold when a second service, a background job, a manual SQL
fix, or an AI-written endpoint writes to the table. Rules enforced only in application code are enforced
only on the paths that happen to run it — and bad data written once persists forever.

**Then draw the database/application line explicitly:**

| Belongs in the database | Belongs in the application |
| :--- | :--- |
| Uniqueness, foreign keys, `NOT NULL`, enum `CHECK`s | Pricing rules, permissions, email sending |
| Monetary precision (`numeric(12,2)`) | Retries, workflow, orchestration |
| Pure derivations as generated columns | Anything product may change weekly |

**The test:** *if a different program wrote this row, would the data be wrong?* If yes, it belongs in the
database.

**The mistake made here:** encoding volatile policy as a `CHECK`. Pricing tiers and feature flags belong
in a table or in code, because changing a `CHECK` is a migration.

**Diagnostic:** a schema with many columns and near-zero `CHECK`/`UNIQUE` was generated without this
step.

---

## Step 7 — Derive indexes from step 3

**Do this mechanically.** For each frequent pattern, build one index ordered: **equality columns first,
then sort columns, then range columns.** Do not add indexes by intuition.

```sql
-- pattern: WHERE project_id = ? AND status <> 'done' ORDER BY position
create index on task (project_id, position) where status <> 'done';

-- pattern: WHERE task_id = ? ORDER BY created_at
create index on comment (task_id, created_at);
```

```js
// MongoDB, same rule (ESR)
db.task.createIndex({ project_id: 1, position: 1, due_at: 1 })
```

**Why order, not set:** an index only limits the scan for equality on its leading columns plus one
inequality on the first unconstrained column. Deriving from the pattern list yields few correct indexes;
guessing yields many redundant ones that slow every write and are never used.

**Two checks needing no runtime data:**

1. Every foreign key column should be the **leading** column of some index. Postgres and SQLite do not
   index foreign keys automatically; MySQL/InnoDB does.
2. An index whose columns are a prefix of another index's columns, in the same order, is redundant.

**The mistake made here:** indexing below ~10k rows. A sequential scan is faster there; the index is
harmless but pointless.

Full detail in [indexes.md](indexes.md).

---

## Step 8 — Validate, then deliver

### Walk every access pattern through the finished design

On paper, before writing any application code. For each pattern from step 3: write the query, **name the
index it uses**, and count the round trips.

```
Pattern: "Open project board"
  SELECT id, title, position FROM task
   WHERE project_id = $1 AND status <> 'done' ORDER BY position LIMIT 50;
  Index: task(project_id, position) WHERE status <> 'done'    OK
  Round trips: 1                                              OK

Pattern: "Task detail + comments + author names"
  1 query + join to user                                      OK
  (not 1 + N comment-author lookups)
```

**This is the only step that catches a shape which satisfies every constraint but cannot answer a
required question cheaply.** It reliably surfaces missing indexes, patterns needing a column that lives
on the wrong table, and list-plus-detail shapes that will become N+1 queries.

A sequential scan is the correct plan for small tables and for genuine analytics — do not add an index
to make an `EXPLAIN` look tidier.

### Plan the first migration as a file

One migration = one deployable unit, forward-only. The first migration sets the pattern every later one
copies.

```sql
ALTER TABLE t ADD COLUMN a text DEFAULT 'x';       -- PG 11+: no rewrite
ALTER TABLE t ADD CONSTRAINT c CHECK (...) NOT VALID;
ALTER TABLE t VALIDATE CONSTRAINT c;               -- SHARE UPDATE EXCLUSIVE only
CREATE INDEX CONCURRENTLY ...;                     -- never inside a transaction
```

**On an empty or pre-launch database all of this is unnecessary ceremony — just write the DDL.** The
concurrency-safe forms matter only once real traffic writes to the table. Full detail in
[migrations.md](migrations.md).

### Record the decisions and the omissions

One short document beside the schema:

```markdown
- comment.author_id nullable: users can be deleted, comments stay
- comments referenced not embedded: unbounded per task
- no soft delete on membership: revoking access must be real
- NOT modelled yet: attachments, notifications, audit log
- open: does a task move between projects? (assumed no; FK is
  ON DELETE CASCADE, revisit if yes)
```

**Why:** six weeks later nobody remembers why `author_id` is nullable, so the next change — often made by
an AI with no context — violates an invariant that was never written down.

**Keep it to decisions and exclusions**, never a column-by-column restatement of the schema, which will
drift and mislead. A well-named schema with `CHECK` constraints already documents most of itself; only
the reasoning needs writing down. In Postgres, `COMMENT ON TABLE`/`COLUMN` keeps it in the database
itself.

---

## What to ask the user

You cannot infer these. Ask before starting, not after delivering:

- **Scale.** Rows today and in a year, for the two or three biggest entities.
- **Ownership.** Who owns each record, and is there a tenant/organisation boundary?
- **Deletion.** What has to be recoverable, and what must be genuinely erasable?
- **Uniqueness.** What must never be duplicated, in the user's own words.
- **Concurrency.** Anything where two people acting at once must not both succeed — stock, seats,
  invites, balances.
- **Engine.** If not already chosen: is anything already in place, and is there a hosting constraint?

If they do not know, say what you assumed and mark it in the decisions document as open.

---

## Sources

MongoDB: [schema-design-process](https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/),
[identify-workload](https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/identify-workload/),
[equality-sort-range](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/).
PostgreSQL: [indexes-multicolumn](https://www.postgresql.org/docs/current/indexes-multicolumn.html),
[sql-altertable](https://www.postgresql.org/docs/current/sql-altertable.html).
