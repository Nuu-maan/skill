# Review report template

Follow this structure. Adjust the amount of detail to the size of the review, but keep the order: the
verdict comes first, the reasoning after.

---

## Verdict

One or two sentences. Lead with whether this is safe to ship or to grow.

> **Two things will bite you before you have real users.** Your `total` column is a float, so order
> totals will drift by cents, and none of your foreign keys are indexed, so deleting a user gets slower
> every week.

or

> **The schema is sound.** Three things will matter at around 100k rows; nothing is wrong today.

Then the counts:

| Severity | Count |
| :--- | :--- |
| Critical | 1 |
| High | 3 |
| Medium | 4 |
| Low | 2 |

---

## What I looked at

Two or three lines. This tells the user how much weight to put on the result.

> Postgres 15 via Supabase, Prisma 5. Reviewed `prisma/schema.prisma`, all 14 migrations in
> `prisma/migrations/`, and the queries in `src/server/`. I did not connect to the database, so row
> counts and the indexes that actually exist in production are unverified. RLS policies are not in the
> repo — I could not review them.

State explicitly what you did **not** cover.

---

## Findings

One block per finding, ordered Critical → Low. Do not merge distinct issues, and do not split one issue
across several entries.

---

### C1 · Order totals will drift by cents

**Severity:** Critical
**Where:** `prisma/schema.prisma:41`

```prisma
model Order {
  total Float
}
```

**What happens:** `Float` becomes `double precision` in Postgres. Binary floats cannot represent `19.99`
exactly, so every total is stored slightly off. Summing a hundred orders drifts by a cent or two, and
`WHERE total = 19.99` will miss rows that display as $19.99. It is invisible with test data and shows up
the first time you reconcile against Stripe.

**Fix:**

```prisma
total Decimal @db.Decimal(12, 2)
```

**Before you run it:** the existing values are already imprecise, so the migration cannot recover the
true amounts. Decide first whether to round on conversion or recompute from the line items. That is a
product decision, not a technical one.

**Also affects:** `OrderLine.unitPrice:52`, `Refund.amount:78` — same type.

---

### H1 · No index on any foreign key

**Severity:** High
**Where:** `prisma/schema.prisma` — `Order.userId:39`, `OrderLine.orderId:50`, `Comment.postId:63`

```prisma
model Order {
  userId String
  user   User @relation(fields: [userId], references: [id])
  // no @@index([userId])
}
```

**What happens:** Postgres does not create an index for foreign key columns, and Prisma does not either —
despite what Prisma's own docs suggest. So `WHERE userId = ?` scans the whole orders table, and deleting
one user scans every child table once per row to check for references, while holding locks. At a few
hundred thousand rows a delete goes from milliseconds to minutes and blocks other writers.

**Fix:**

```prisma
model Order {
  userId String
  user   User @relation(fields: [userId], references: [id])
  @@index([userId])
}
```

Generate the migration and edit it to use `CONCURRENTLY`, one statement per file — Prisma wraps
migrations in a transaction and `CONCURRENTLY` cannot run inside one.

```sql
CREATE INDEX CONCURRENTLY "Order_userId_idx" ON "Order"("userId");
```

**How to check it yourself:** after applying, run

```sql
EXPLAIN ANALYZE SELECT * FROM "Order" WHERE "userId" = '<some id>';
```

It should say `Index Scan`, not `Seq Scan`.

---

*(continue for each finding)*

---

## Needs your decision

Anything you cannot decide alone. Give the options and a recommendation, not an open question.

- **Duplicate emails.** Adding `@unique` to `User.email` will fail — there are already duplicates from
  before you added the check in application code. Run
  `SELECT email, count(*) FROM "User" GROUP BY 1 HAVING count(*) > 1;` and tell me how many. If it is a
  handful, I would merge them by hand; if it is hundreds, we need a rule for which account wins.
- **Soft deletes.** You have `deletedAt` on `User` but the unique index on `email` ignores it, so a
  deleted user can never re-register. Do you want re-registration to work? If yes, the fix is a partial
  unique index and I can write it.

---

## Could not verify

Be explicit rather than silent.

- **Row counts.** Everything about lock duration depends on table size. Run
  `SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC LIMIT 10;`
- **Which indexes exist in production.** The repo shows what migrations intend, not what is there.
  `SELECT indexname, indexdef FROM pg_indexes WHERE schemaname = 'public';`
- **RLS policies.** Not in the repo. If you set them in the Supabase dashboard, I cannot see them —
  and if you have not set any, every table is readable by anyone with the anon key.

---

## Next step

Close with a specific offer.

> I can start with H1 — that is three `@@index` lines plus one migration file, no data risk, and it is
> the change with the biggest effect. C1 I would rather not touch until you have decided how to handle
> the existing values. Want me to do the indexes now?

---

## Writing rules

- **Concrete outcome, not category name.** "Deleting a user gets slower every week and eventually times
  out", not "missing index on foreign key". Name the category afterwards if useful.
- **Quote the actual definition**, with a real file and line. Never paraphrase what you think is there.
- **Show the fixed code**, not a description of the fix.
- **Always say whether a migration locks**, which table, and roughly for how long — or say you cannot
  know without the row count.
- **One "also affects" line** when the same pattern repeats, rather than a separate finding per column.
- **No severity inflation.** A missing index on a 200-row lookup table is Low. If everything is Critical,
  nothing is.
- **No filler findings.** If there are two issues, report two.
- **Give a self-check.** For each fix, one command the user can run to see for themselves that it worked.
