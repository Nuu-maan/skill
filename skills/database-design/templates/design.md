# Design proposal template

Follow this structure when delivering a new schema. The order matters: the user needs to see that you
understood the problem before they can judge the solution.

Do not open with DDL. A wall of `CREATE TABLE` gives the reader no way to tell whether you understood
what they asked for.

---

## What I understood

Restate the feature in your own words, as concrete actions. This is where a misunderstanding gets caught
cheaply — three lines here saves a rewrite later.

> A user creates projects. They invite other users to a project as either an editor or a viewer. Inside a
> project there are tasks, tasks have comments, and comments are written by users. Tasks can be reordered
> by dragging.

Then state what you assumed, because you had to:

> **Assumed:** a task belongs to exactly one project and never moves between them; a project has one
> owner who cannot be removed; comments are never edited after posting. Tell me if any of those is wrong
> — the first one in particular changes the foreign key.

---

## Access patterns

The table from step D3. **Lead with this, not with the schema** — it is what the schema is derived from,
and it is the part the user can actually check.

| Action | R/W | Filter | Sort | Rough frequency |
| :--- | :--- | :--- | :--- | :--- |
| Open project board | R | `project_id`, `status <> 'done'` | `position` | very hot |
| Task detail with comments | R | `task_id` | `created_at` | hot |
| Mark task done | W | `task_id` | — | hot |
| List my projects | R | `user_id` | `updated_at desc` | hot |
| Admin: overdue tasks | R | `due_at < now()` | `due_at` | rare |

> If a screen you need is missing from this table, say so — every index below comes from this list.

---

## The schema

Give it as runnable DDL, or in the project's ORM syntax if one is already in use. Comment the decisions,
not the obvious.

```sql
CREATE TABLE project (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id   uuid NOT NULL REFERENCES app_user(id) ON DELETE RESTRICT,
  name       text NOT NULL CHECK (length(trim(name)) > 0),
  created_at timestamptz NOT NULL DEFAULT now(),
  deleted_at timestamptz,
  UNIQUE (owner_id, name)
);

CREATE TABLE task (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id uuid NOT NULL REFERENCES project(id) ON DELETE CASCADE,
  title      text NOT NULL,
  status     text NOT NULL DEFAULT 'todo'
             CHECK (status IN ('todo','doing','done')),
  position   numeric NOT NULL,        -- fractional: insert between neighbours
  done_at    timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  CHECK ((status = 'done') = (done_at IS NOT NULL)),
  UNIQUE (project_id, position)
);
```

---

## Indexes

Each one paired with the access pattern that justifies it. **An index with no pattern beside it should
not exist.**

```sql
-- "Open project board"
CREATE INDEX ON task (project_id, position) WHERE status <> 'done';

-- "Task detail with comments"
CREATE INDEX ON comment (task_id, created_at);

-- "List my projects" — plus every FK needs its child-side index on Postgres
CREATE INDEX ON project (owner_id);
CREATE INDEX ON membership (user_id);
CREATE INDEX ON task (project_id);
```

---

## Walking the patterns through it

Prove the design answers each pattern. This is the step that catches a schema which satisfies every
constraint but cannot answer a required question cheaply.

> **Open project board**
> ```sql
> SELECT id, title, position FROM task
>  WHERE project_id = $1 AND status <> 'done'
>  ORDER BY position LIMIT 50;
> ```
> Uses `task (project_id, position) WHERE status <> 'done'`. One query, no sort step.
>
> **Task detail with comments and author names**
> One query joining `comment` to `app_user`. Not 1 + N author lookups.
>
> **List my projects**
> Joins `membership` to `project` on the indexed `user_id`. One query.

---

## Decisions and what I left out

The document that makes the schema reviewable in six weeks. Keep it to reasoning and exclusions — never a
column-by-column restatement.

- `comment.author_id` is nullable with `ON DELETE SET NULL`: users can be deleted, their comments stay.
- `project.owner_id` is `ON DELETE RESTRICT`, not CASCADE: deleting a user should fail loudly rather than
  silently destroy their team's projects.
- `position` is `numeric`, not `integer`: dragging a task between two others is then one `UPDATE`
  instead of renumbering every row after it.
- Soft delete on `project` only. Tasks and comments hard-delete with their project.
- **Not modelled yet:** attachments, notifications, activity feed, per-project roles beyond
  editor/viewer.
- **Open question:** can a task move between projects? I assumed no, so `project_id` is
  `ON DELETE CASCADE`. If yes, that changes and we need to think about `position` across the move.

---

## First migration

State whether the sequence is safe to run against an existing database, or is a greenfield create.

> This is a greenfield schema, so it is one migration and none of the concurrency ceremony applies —
> plain `CREATE INDEX` is correct and faster here. Once this table has real traffic, later index
> additions need `CONCURRENTLY` in their own migration file.

---

## Writing rules

- **Understanding first, DDL second.** If the restatement is wrong, nothing after it matters.
- **Every index cites its access pattern.** No orphan indexes.
- **Every assumption is stated as an assumption**, in a place the user will read.
- **Comment the decisions, not the syntax.** `-- fractional: insert between neighbours` earns its line;
  `-- primary key` does not.
- **Name what you did not model.** An omission the user notices themselves reads as an oversight; one you
  named reads as a decision.
- **Do not over-engineer for scale you were not told about.** Partitioning, sharding, and archival tiers
  are premature below tens of millions of rows. If you think they will be needed, say when — do not build
  them now.
