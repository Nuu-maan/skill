# BaaS — Supabase, Firebase, Appwrite, PocketBase

Load whenever the frontend talks to a database or auth provider directly, with little or no server
code between them.

**This is the highest-risk setup for a frontend developer.** The provider moves the trust boundary
into the database itself. If the security rules are not written — and by default they often are not,
or are set to "allow everything for 30 days" — the entire database is readable and writable by
anyone who opens devtools and copies the URL and key out of the JavaScript bundle. There is no
server-side code to review, so nothing looks wrong.

Establish this first, before anything else.

## Contents

- The one question that matters
- Supabase
- Firebase
- Appwrite and PocketBase
- Storage
- Serverless functions
- What to verify by testing

---

## The one question that matters

**Can an anonymous client read or write data it should not?**

Everything else is secondary. Answer it by finding the rules:

| Provider | Where the rules live |
| :--- | :--- |
| Supabase | Postgres row-level security policies, per table |
| Firebase | `firestore.rules`, `database.rules.json`, `storage.rules` |
| Appwrite | Per-collection and per-document permissions |
| PocketBase | Per-collection API rules |

If those files do not exist in the repo, the rules were configured in a dashboard and cannot be
reviewed from the code. Say so explicitly and ask the user to check, rather than assuming they are
fine. Missing rules is the default state, not the exception.

---

## Supabase

**Row-level security.** RLS is **off by default** on a new table. With RLS off and the anon key in
the browser, the table is fully readable and writable by anyone.

Look for `ALTER TABLE x ENABLE ROW LEVEL SECURITY;` in the migrations for every table. A table with
no `ENABLE ROW LEVEL SECURITY` is a Critical finding. So is a policy of the form:

```sql
CREATE POLICY "Enable all" ON profiles FOR ALL USING (true);            -- everyone, everything
CREATE POLICY "public read" ON orders FOR SELECT USING (true);          -- everyone reads all orders
```

Correct shape:

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY "own orders: read" ON orders
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "own orders: insert" ON orders
  FOR INSERT WITH CHECK (auth.uid() = user_id);
```

**`USING` vs `WITH CHECK`.** `USING` filters rows that can be read or targeted; `WITH CHECK`
validates rows being written. A policy with only `USING` on an `UPDATE` lets a user modify their own
row into someone else's — set `user_id` to another uuid and it is no longer theirs. Every
`INSERT`/`UPDATE` policy needs `WITH CHECK`.

**Separate policies per operation.** `FOR ALL` is coarse; write `SELECT`, `INSERT`, `UPDATE`, and
`DELETE` policies deliberately. Users frequently should not be able to `DELETE` at all.

**The service role key bypasses RLS entirely.** It must exist only on the server. In the browser it
is total database compromise. Grep for:

```
NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY
VITE_SUPABASE_SERVICE_ROLE
service_role
SUPABASE_SERVICE_KEY   (in any client-side file)
```

The `anon` key **is** meant to be public — it is safe only because RLS constrains it. That is the
whole security model: anon key + RLS. Without RLS, the anon key is a database password.

**Columns are not covered by RLS.** RLS controls rows, not columns. A `profiles` table readable by
all authenticated users exposes every column in it — including an email or phone number you meant to
keep private. Use a view with only the public columns, and grant on that.

**Postgres functions.** `SECURITY DEFINER` functions run as the owner and bypass the caller's RLS.
Every one of them needs its own authorization check inside, and an explicit `search_path` set.
`SECURITY INVOKER` is the safer default.

**Realtime.** Subscriptions respect RLS only if RLS is enabled on the table. Check which tables are
in the realtime publication.

**Storage.** Buckets have their own policies. A public bucket is world-readable by URL — fine for
avatars, wrong for invoices or ID documents.

**Auth.** Confirm email confirmation is required if your logic assumes verified emails, and that
sign-ups are not open if the app is meant to be invite-only. Check redirect URL allowlists.

---

## Firebase

**Test mode rules are time-limited and fully open:**

```
allow read, write: if request.time < timestamp.date(2025, 1, 1);
```

After the date, everything breaks; before it, everything is public. Either way it is a finding.
`allow read, write: if true;` is the same thing without the expiry.

**`request.auth != null` is not authorization.** It means "any signed-in user", and with anonymous
auth or open sign-up that is effectively everyone. Ownership needs the resource:

```
match /orders/{orderId} {
  allow read:   if request.auth != null && resource.data.userId == request.auth.uid;
  allow create: if request.auth != null && request.resource.data.userId == request.auth.uid;
  allow update: if request.auth != null && resource.data.userId == request.auth.uid
                && request.resource.data.userId == resource.data.userId;   // cannot reassign
  allow delete: if false;
}
```

**`resource` vs `request.resource`.** `resource.data` is the existing document; `request.resource.data`
is the incoming one. Checking only `resource` on a create is meaningless (there is no existing doc),
and checking only `request.resource` on an update lets a user overwrite a document they do not own.
Both are extremely common mistakes.

**Rules do not cascade the way people expect.** A rule on `/users/{uid}` does not protect
`/users/{uid}/private/{doc}` unless you use a recursive wildcard (`{document=**}`) or write rules for
the subcollection. Check every subcollection has coverage.

**Validate writes in the rules.** Rules are the only server-side validation in a pure-Firebase app.
Constrain field types, sizes, and allowed keys — otherwise a client writes `role: "admin"` or a 5 MB
string into a document.

**Client-side security is not security.** Hiding a button, filtering a query in JavaScript, or
checking a role in the React app does nothing — the client can issue any query the rules permit.

**The Firebase config object in the frontend is public by design** (apiKey, projectId). That is
expected. The service account JSON is not — it grants admin access and bypasses all rules. It must
never be in the repo or the bundle. Grep for `"type": "service_account"` and `private_key`.

**Admin SDK in a Cloud Function bypasses rules**, so every function must do its own authorization on
`context.auth`. An HTTP function is a public endpoint unless it checks a token.

**Cost.** Firestore bills per document read. A rule that permits a broad query lets anyone iterate
your collection and generate a large bill. Rate limiting is on you (App Check, or a function
gateway).

---

## Appwrite and PocketBase

**Appwrite:** permissions are set per collection and optionally per document. A collection with
`Any` read is public. Document-level security must be explicitly enabled on the collection, otherwise
collection-level permissions apply to every document uniformly. API keys are server-side only —
never in client code.

**PocketBase:** each collection has List/View/Create/Update/Delete API rules. An **empty** rule means
"anyone" for that operation; a rule of `null` (locked) means admin-only. Empty is the dangerous
default to look for. Ownership rules look like `@request.auth.id = user`. Verify the Create rule also
constrains the owner field (`@request.data.user = @request.auth.id`), otherwise a user creates records
attributed to someone else. Admin UI must not be publicly reachable without a strong password.

---

## Storage

For every provider:

- Public buckets are readable by anyone with the URL, and URLs leak — in referrers, screenshots, and
  shared links. Anything private belongs in a private bucket served via time-limited signed URLs.
- Write access to storage means an attacker can fill it (cost) or upload content served from your
  domain (stored XSS if the file is HTML or SVG and served inline).
- Constrain uploads by size and content type in the rules, not only in the client.
- Path structure matters: `avatars/{userId}/...` with a rule tying the path segment to the caller's
  uid is how you scope writes.

---

## Serverless functions

Supabase Edge Functions, Firebase Cloud Functions, Appwrite Functions:

- An HTTP-triggered function is a **public endpoint**. Verify the caller's token inside the function;
  do not assume it is only called from your app.
- Functions typically run with a privileged key that bypasses rules. That makes every function a
  potential bypass of your entire security model — each one needs its own authorization check.
- Validate all input. The function's arguments come from the client.
- Secrets go in the platform's function secret store, not in the repo.

---

## What to verify by testing

Reading rules is not enough — they are hard to reason about. Ask the user to run these, or run them
if credentials for a test project are available:

1. **Anonymous read.** With no session, query each significant table/collection. Anything that
   returns rows is public. Compare against what should be public.
2. **Anonymous write.** Attempt an insert into a table you expect to be protected.
3. **Cross-user read.** Sign in as user A, query for user B's rows by id. Expect empty.
4. **Cross-user write.** As user A, attempt to update one of B's rows, and attempt to create a row
   with `user_id` set to B.
5. **Column exposure.** Select `*` from any table the client can read and check what comes back.
6. **Ownership reassignment.** As A, update your own row setting the owner field to B.

Test 4 and test 6 are the ones most often missed and most often exploitable.
