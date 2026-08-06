# Fixing safely

Load in Phase 6, after the user approves fixes. The goal is that every change is small, correct,
verifiable, and explained well enough that the user could have written it themselves.

## Contents

- Rules
- Order of work
- The loop for each fix
- Explaining a fix
- Fix patterns that are usually safe
- Fixes that need a conversation first
- Verification
- When you cannot verify

---

## Rules

1. **One concern per change.** Do not fix an IDOR and reformat the file. The user must be able to
   read the diff and understand exactly what changed and why.
2. **Smallest correct change.** Add the ownership filter; do not restructure the repository layer.
3. **Never silently widen scope.** If a fix requires a change the user did not approve — a new
   dependency, a schema migration, an environment variable, a behavior change — stop and say so
   before making it.
4. **Preserve behavior for legitimate users.** A fix that breaks the happy path will be reverted, and
   with it the security improvement.
5. **Do not fix what you did not report.** Anything new you notice while editing goes in a follow-up
   list, not into the diff.
6. **Never commit or push** unless asked.

---

## Order of work

Fix in severity order, but within a severity, prefer changes that are self-contained.

1. **Critical** first, one at a time, verified between each.
2. **Structural fixes before local ones** when a single change removes a whole class of bug — adding
   a global auth guard with an explicit public allowlist is better than patching eleven handlers, and
   it prevents the twelfth.
3. **High**, same discipline.
4. **Medium and Low** can be batched by theme (all timeouts, all response allowlists), because they
   are individually low-risk and reviewing them together is easier.

If a Critical fix needs a migration or a dependency, do the code-only Criticals first so the user has
something deployable immediately.

---

## The loop for each fix

```
Per fix:
- [ ] Re-read the current code (it may have changed)
- [ ] Make the change
- [ ] Check for the same pattern elsewhere in the codebase
- [ ] Verify (tests / typecheck / lint / run it)
- [ ] Report what changed in one or two lines
```

**The "same pattern elsewhere" step is the one that gets skipped and matters most.** If one handler
was missing an ownership check, grep for the same query shape across the codebase — it is almost
never a single occurrence. Fix all of them or list the ones you did not.

---

## Explaining a fix

The user is learning backend from this. For each fix, give three short lines — no more:

> **What I changed:** `orders.ts:34` now queries by `{ id, userId }` instead of `{ id }`.
> **Why:** the id comes from the URL, so without the `userId` filter any logged-in user could read
> any order by changing the number.
> **How to check it yourself:** log in as one account, note an order id, log in as another and
> request that id — you should get a 404.

The third line matters most. It turns a change they must trust into one they can confirm.

---

## Fix patterns that are usually safe

These rarely break the happy path and can be applied confidently once approved:

- Adding an ownership filter to a query (returns 404 instead of another user's data).
- Replacing string-concatenated SQL with parameterized queries.
- Replacing `exec` with `execFile`/argv form.
- Adding a validation schema that mirrors the fields the handler already uses.
- Replacing a field spread with an explicit allowlist of the fields already being written.
- Adding timeouts to outbound calls.
- Adding cookie flags (`httpOnly`, `secure`, `sameSite: 'lax'`).
- Removing a stack trace from an error response and logging it instead.
- Replacing an explicit output object for `res.json(model)`.
- Adding `.gitignore` entries and moving hardcoded config to environment variables **with validation
  at boot** — but see below about rotation.
- Adding indexes on foreign keys (do it concurrently on a large table).

---

## Fixes that need a conversation first

Do not apply these without explaining the consequence and getting a yes:

| Fix | Consequence to state |
| :--- | :--- |
| Rotating a leaked secret | The old key stops working the moment it is revoked; anything still using it breaks. Sequence: issue new → deploy → revoke old. |
| Changing a password hashing algorithm | Existing hashes cannot be converted. Requires rehash-on-next-login, keeping both verifiers during the transition. |
| Adding CSRF protection | Every existing client must send the token. Breaks any integration not updated. |
| Tightening CORS | Any origin not on the list stops working, including staging and local dev. |
| Adding `NOT NULL` or a unique constraint | Fails if existing rows violate it. Find and fix the rows first. |
| Adding auth to a previously public endpoint | Breaks any consumer relying on it. Find out who calls it. |
| Adding rate limits | Legitimate heavy users may be blocked. Start generous, watch, tighten. |
| Any schema migration | Needs a rollback plan and, on a large table, a lock-safe strategy. |
| Switching session storage | Logs everyone out. |
| Upgrading a major dependency version | Out of scope for a security fix unless it is the fix itself. |

For a leaked secret specifically: rotation is the fix. Removing it from the code without rotating it
fixes nothing, and saying otherwise gives the user false confidence. Be explicit about that.

---

## Verification

After each fix, in this order — use whatever the project actually has:

1. **Typecheck / compile.** `tsc --noEmit`, `mypy`, `go build ./...`, `mvn compile`, `dotnet build`.
2. **Lint.** Catches an unused import or an unreachable branch left by an edit.
3. **Tests.** Run the existing suite. If a test now fails, decide honestly whether the test encoded
   the bug — say which, do not just change the test to pass.
4. **Targeted check.** For a security fix, the negative case is the point: confirm the unauthorized
   request is now rejected, not only that the authorized one still works.
5. **Run it** if the project can be started, and exercise the changed route.

Report failures accurately. If the suite was already failing before your change, say that.

---

## When you cannot verify

Say so, precisely, rather than implying the fix is confirmed:

> I applied the ownership filter to all 7 handlers. I could not run the test suite — `npm test` is
> not defined in `package.json` — so this is verified by reading only. Before deploying, check
> `/api/orders/:id` returns 404 for an order belonging to another account.

Then give the user a short manual test list for the fixes you made. An unverified fix that is
labelled unverified is useful. An unverified fix presented as done is worse than no fix.

---

## Finishing

Close with:

- What was fixed, grouped by severity.
- What was **not** fixed and why (needs a decision, needs a migration, out of scope).
- Anything the user must do outside the code: rotate a key, set an environment variable in the
  hosting dashboard, enable a provider setting, run a migration.
- The manual checks they should run before deploying.
