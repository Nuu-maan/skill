# Fixing safely

How to apply database fixes without breaking things. Load before making any change.

**This domain is different from application code.** A bad code change throws an error you can revert. A
bad migration deletes a column, and no amount of reverting brings the data back. Treat every fix here as
one-way until proven otherwise.

## Contents

- The seven rules
- Order of work
- The loop for each fix
- Explaining the change
- Usually safe to just do
- Needs a conversation first
- Verifying
- When you cannot verify

---

## The seven rules

1. **One concern per change.** One migration should do one thing. If it fails, you need to know which
   part failed.
2. **Smallest correct change.** Add the index. Do not also rename the table because the name bothers you.
3. **Never silently widen scope.** If fixing the reported problem requires touching something else, say
   so before doing it.
4. **Preserve behaviour.** A fix that changes what the application returns is a feature change. Flag it.
5. **Do not fix what you did not report.** The user approved a list. Anything outside it needs a new
   sentence, not a quiet edit.
6. **Say whether it locks, and for how long.** Every schema change, every time. If you do not know the
   table size, say that too.
7. **Never run a destructive statement.** `DROP`, `TRUNCATE`, and un-batched `UPDATE`/`DELETE` are the
   user's to run, after they have a backup. Write them out; do not execute them.

---

## Order of work

Fix in this order, not in severity order:

1. **Things that are losing or corrupting data right now.** A money column in a float keeps drifting
   while you discuss it.
2. **Things that block the next deploy.** A migration that will lock a table.
3. **Correctness bugs** — the wrong-results class: `NOT IN` with NULLs, a missing unique constraint,
   `DISTINCT` over a fan-out join.
4. **Scale problems** — missing indexes, N+1, unbounded queries.
5. **Hygiene.**

Within that, do the **reversible** ones first. Adding an index is reversible. Changing a column type is
not.

---

## The loop for each fix

For every single fix:

1. **Re-read the current definition.** Not your notes from the audit — the file, now. A later migration
   may already have addressed it.
2. **Write the change.** A migration file, not a live edit, unless the project genuinely has no migration
   tool.
3. **State the lock.** Which lock, on which table, for roughly how long, and whether reads or writes are
   affected.
4. **Check the same pattern elsewhere.** If `orders.user_id` had no index, check every other foreign key.
   **This is the step that gets skipped**, and it is the one that turns a fix into a real improvement.
5. **Say what you could not check.** Row counts, production configuration, indexes that exist in the
   database but not in the repo.

---

## Explaining the change

Three lines per fix. No more.

> **What I changed:** Added an index on `orders.user_id`.
>
> **Why:** Postgres does not index foreign key columns for you, so every time you delete a user it
> scanned the whole orders table to check for referencing rows.
>
> **How to check it yourself:** Run `EXPLAIN ANALYZE DELETE FROM users WHERE id = 1;` in a transaction
> you roll back. It should say `Index Scan` on orders, not `Seq Scan`.

The third line matters most for this audience. **Give them a way to see it worked that does not require
trusting you.**

---

## Usually safe to just do

Once approved, these rarely need further discussion:

- Adding an index **concurrently** on Postgres, or online on MySQL.
- Adding a **nullable** column with no default.
- Adding a `CHECK` or foreign key as `NOT VALID`, then validating separately.
- Adding a missing `@@index` / `db_index` / `index()` declaration in an ORM schema.
- Rewriting a query: `NOT IN` → `NOT EXISTS`, `OFFSET` → keyset, `SELECT *` → explicit columns.
- Adding `select_related` / `include` / `join fetch` to kill an N+1.
- Adding `AsNoTracking`, `.lean()`, or `runValidators` on the paths where they are unambiguously right.
- Setting `PRAGMA foreign_keys = ON` in SQLite connection setup.
- Adding `lock_timeout` to the migration runner.

---

## Needs a conversation first

Never do these without explicit agreement on the specific plan:

| Change | Why it needs a decision |
| :--- | :--- |
| Changing a money column from float to numeric | Existing values are already imprecise. Someone must decide whether to round, recompute from source, or accept the drift. |
| Adding `NOT NULL` to an existing column | There are already NULL rows. What happens to them is a product question. |
| Adding a unique constraint | There are already duplicates. Which one wins is a product question. |
| Dropping or renaming a column | Irreversible, and needs expand-and-contract across three deploys. |
| Changing a column type | Full table rewrite on most engines, plus a possible application-level behaviour change. |
| Changing a primary key type | Touches every foreign key, every cached id, and every external URL. |
| Adding `ON DELETE CASCADE` | Changes what a delete destroys. Get explicit confirmation of the blast radius. |
| Dropping an "unused" index | `idx_scan = 0` needs a full business cycle of statistics. Never drop a unique or FK-supporting index on that evidence. |
| Converting an enum to text, or vice versa | Itself a rewriting migration. |
| Enabling `additionalProperties: false` on a live MongoDB validator | Will reject writes during a rolling deploy. |
| Turning on `strictQuery`, `open-in-view: false`, or global `AsNoTracking` | Silent behaviour changes across the whole app. |
| Anything requiring a backfill | Batch size, timing, and whether it runs in or out of the migration are all decisions. |

For each of these, present the options and a recommendation — do not present a menu and wait.

---

## Verifying

In this order:

1. **Does the migration apply?** On a copy, or a fresh database from migrations.
2. **Does it reverse?** If there is a `down`, run it. If there cannot be one, say so plainly rather than
   writing a fake.
3. **Does the query plan change?** For an index fix, this is the whole point. Ask for
   `EXPLAIN (ANALYZE, BUFFERS)` before and after.
4. **Do the tests pass?** If there are any.
5. **Is the same pattern fixed everywhere?** Re-run the grep that found it.

For a Postgres index built concurrently, always check it is valid afterwards:

```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
```

A failed `CONCURRENTLY` build leaves an INVALID index that queries ignore while every write still
maintains it — the worst of both worlds, and silent.

---

## When you cannot verify

Say it. Explicitly, in the same message as the fix.

You cannot connect to the database, so you cannot know:

- how many rows a table has, and therefore how long a lock will hold;
- which indexes actually exist in production, as opposed to in the repo;
- whether a `CHECK` constraint written for MySQL 8 is running against 5.7;
- whether the backfill you just wrote will finish in ten seconds or ten hours.

For each, give the exact statement that answers it and ask for the output. **"Run this and paste the
result" is a better answer than a confident guess.**

Never write "this is safe" about a migration when you do not know the table size. Write "this is safe on
a table under ~100k rows; run `SELECT count(*)` first."
