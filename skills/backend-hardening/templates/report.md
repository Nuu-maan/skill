# Report template

Follow this structure. Adjust the amount of detail to the size of the audit, but keep the order: the
verdict comes first, the reasoning comes after.

---

## Verdict

One or two sentences. Lead with whether it is safe to deploy.

> **Do not deploy yet.** 2 Critical and 4 High issues — most importantly, any logged-in user can read
> every other user's orders, and your Stripe secret key is committed to the repository.

or

> **No Critical or High issues found.** 3 Medium items worth fixing before this gets real traffic.

Then the counts:

| Severity | Count |
| :--- | :--- |
| Critical | 2 |
| High | 4 |
| Medium | 6 |
| Low | 3 |

---

## What I looked at

Two or three lines. This tells the user how much weight to put on the result.

> Stack: Express 4 + Prisma + Postgres, deployed on Railway. Reviewed 23 routes across
> `src/routes/`, the auth middleware, the Prisma schema, and the Dockerfile. I could not run the
> app, so findings are from reading the code. I did not review the frontend.

State explicitly what you did **not** cover.

---

## Findings

One block per finding, ordered Critical → Low. Do not merge distinct issues into one entry, and do
not split one issue across several entries.

---

### C1 · Any logged-in user can read every order

**Severity:** Critical
**Where:** `src/routes/orders.ts:34`

```ts
const order = await prisma.order.findUnique({ where: { id: req.params.id } });
res.json(order);
```

**What happens:** The order id comes from the URL and nothing checks who owns it. Logged in as any
account, changing `/api/orders/1041` to `/api/orders/1042` returns another customer's order —
including their address and the last four digits of their card. Ids are sequential, so the whole
table can be walked in a few minutes with a script.

**Fix:**

```ts
const order = await prisma.order.findFirst({
  where: { id: req.params.id, userId: req.user.id },
});
if (!order) return res.status(404).json({ error: 'Not found' });
```

Return 404 rather than 403 so the response does not confirm that the order exists.

**Also affects:** `src/routes/invoices.ts:22`, `src/routes/tickets.ts:51` — same pattern.

---

### C2 · Stripe secret key committed to the repository

**Severity:** Critical
**Where:** `.env:7` (tracked by git), present since commit `a3f91c2`

```
STRIPE_SECRET_KEY=sk_live_4eC***
```

**What happens:** `.env` is not in `.gitignore`, so the live key is in the repository history. If the
repo is ever public — or already is — this key allows charges, refunds, and reading your full
customer list. Public repositories are scanned continuously and leaked keys are typically abused
within minutes.

**Fix, in this order:**

1. Roll the key in the Stripe dashboard now and set the new one in your host's environment settings.
2. Revoke the old key once the new one is deployed.
3. Add `.env` to `.gitignore` and remove the file from the working tree.
4. Purge it from git history (`git filter-repo`) if the repo is or will be public.

Removing it from the code without rolling it does not help — anyone who cloned the repo already
has it.

---

*(continue for each finding)*

---

## Not fixed / needs your decision

Anything you cannot decide alone.

- **Rate limiting store.** The current limiter keeps counts in memory, so it resets on every deploy
  and does not work across your two instances. Fixing it properly needs Redis — do you have one, or
  should we use the host's built-in rate limiting instead?
- **Adding `NOT NULL` to `orders.user_id`.** There are 12 existing rows with a null `user_id`. We
  need to decide what happens to them before adding the constraint.

---

## Could not verify

Be explicit rather than silent.

- Whether the production `DEBUG` flag is actually false — it is read from an environment variable I
  cannot see. Check your host's dashboard.
- Whether the database is reachable from the public internet. Check the network settings in your
  provider.
- Supabase RLS policies are not in the repo, so I could not review them. Run the anonymous-read test
  in `stacks/baas.md`.

---

## Next step

Close with a specific offer, not an open question.

> I can start on the two Criticals now — that is about 20 lines across 3 files for C1, and for C2 I
> will do the `.gitignore` and code changes, but you need to roll the Stripe key yourself. Want me
> to go ahead?

---

## Writing rules

- **Concrete outcome, not category name.** "Any logged-in user can read every other user's orders",
  not "Broken Object Level Authorization (OWASP API1)". Name the category afterwards if useful.
- **Quote the actual code**, with a real file and line. Never paraphrase what you think is there.
- **Show the fixed code**, not a description of the fix.
- **One "also affects" line** when the same pattern repeats, rather than a separate finding per file.
- **No severity inflation.** A missing index is not High. If everything is Critical, nothing is.
- **No filler findings.** If there are only two issues, report two.
- **Never print a real secret.** Mask it and give the location.
