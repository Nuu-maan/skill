# Authorization

What the caller is allowed to do. Load when the codebase has per-user data, roles, admin routes,
teams, or multi-tenancy.

**This is where AI-generated backends fail most often.** Authentication usually gets written, because
it is visible in the UI. Authorization usually does not, because the happy path works without it.

## Contents

- IDOR / broken object-level authorization
- Missing function-level authorization
- Mass assignment and privilege escalation
- Role checks in the wrong place
- Multi-tenancy leaks
- Client-trusted authorization
- Indirect object exposure
- Verifying the fix

---

## IDOR / broken object-level authorization

**Signal:** A handler reads an identifier from the URL, query, or body and looks the record up by
that identifier alone.

```js
// every one of these is the same bug
const order = await Order.findById(req.params.id);
const doc   = await db.document.findUnique({ where: { id: req.params.id } });
const inv   = await Invoice.objects.get(pk=request.GET['id']);
$post = Post::find($request->id);
```

**Why it breaks:** The id is attacker-controlled. Logged in as user A, change `/api/orders/1041` to
`/api/orders/1042` and read user B's order. With sequential ids the whole table is enumerable in
minutes. This is the most common serious flaw in this class of code and it is usually Critical or
High depending on what the object holds.

**Fix:** Scope every lookup to the authenticated principal in the same query. Not a check afterwards
— in the query, so there is no path where it can be skipped.

```js
const order = await Order.findOne({ _id: req.params.id, userId: req.user.id });
if (!order) return res.status(404).json({ error: 'Not found' });
```

```python
order = Order.objects.filter(pk=pk, user=request.user).first()
if order is None:
    raise Http404
```

Return **404, not 403**, for objects the caller does not own. 403 confirms the object exists, which
leaks the id space.

**Where to look beyond the obvious GET:**
- `PATCH`/`PUT`/`DELETE` handlers — a write IDOR is worse than a read IDOR.
- Nested resources: `/teams/:teamId/members/:id` must check the caller belongs to `teamId` **and**
  that `:id` belongs to `teamId`. Checking only one is still exploitable.
- File downloads keyed by id or filename.
- Anything that takes an id in the request **body** rather than the path — easy to miss when scanning.
- Bulk endpoints: `DELETE /items` with `{ids: [...]}` must filter the whole list by owner, not just
  check the first element.
- Background jobs and webhooks that re-fetch by id after the request ended.

---

## Missing function-level authorization

**Signal:** An admin, internal, or privileged route with an authentication check but no role check.
Or a route registered outside the block where the guard middleware is applied.

```js
app.use('/api', requireAuth);          // auth for everything under /api
app.delete('/api/users/:id', deleteUser);   // ...but any logged-in user can call it
app.get('/admin/metrics', getMetrics);      // ...and this is outside /api entirely
```

**Why it breaks:** Any registered user reaches functionality meant for staff. "It is not linked in
the UI" is not access control.

**Fix:** Enumerate every route. Build the table explicitly — do not eyeball it:

| Method + path | Auth required | Role required | Enforced at (file:line) |
| :--- | :--- | :--- | :--- |

Any row with a blank enforcement cell is a finding. Prefer **deny by default**: a global guard that
requires auth, with an explicit allowlist of public routes, rather than opt-in protection per route.

Check that route registration order actually places the guard before the handler, and that a router
mounted at a different prefix has not escaped the guard.

---

## Mass assignment and privilege escalation

**Signal:**

```js
await User.update(req.params.id, req.body);
Object.assign(user, req.body);
const user = new User({ ...req.body });
```
```python
User.objects.create(**request.data)
serializer = UserSerializer(user, data=request.data)   # with fields = '__all__'
```
```php
$user->update($request->all());          // with $guarded = []
```
```ruby
User.new(params[:user])                  # without strong parameters
```

**Why it breaks:** The client controls which columns get written. A signup form that posts
`{email, password, role: "admin"}` creates an admin. A profile update that posts
`{name: "x", credits: 999999, emailVerified: true}` grants itself whatever it likes.

**Fix:** Allowlist fields explicitly at the boundary. Never pass a request body straight into a model.

```js
const { name, bio } = req.body;              // pick, do not spread
await User.update({ id: req.user.id }, { name, bio });
```

Use a schema validator with unknown-key stripping (`zod` with `.strict()`, `pydantic` models,
Laravel `$request->validated()`, Rails strong parameters, DRF explicit `fields`). Confirm the
allowlist actually excludes `role`, `isAdmin`, `permissions`, `verified`, `credits`, `balance`,
`plan`, `organizationId`, `userId`, `createdAt`, and the primary key.

**Related signal:** the user id taken from the body instead of the session.

```js
await Order.create({ userId: req.body.userId, ... });   // client picks whose order this is
await Order.create({ userId: req.user.id,   ... });     // correct
```

Any identity value must come from the verified session or token, never from the request payload.

---

## Role checks in the wrong place

**Signal:** Role read from a JWT claim, a cookie, a header, or `localStorage` echoed back by the
client, and trusted without a server-side lookup. Or a role check inside the frontend only.

**Why it breaks:** If the client can influence the value, it is not a permission. A JWT claim is
safe only if the token signature is verified **and** the claim was set by the server at issue time —
but it goes stale, so a demoted user keeps their role until the token expires.

**Fix:** For anything high-impact (billing, deletion, admin actions), read the role from the database
at request time. For lower-impact checks, a verified JWT claim is acceptable if token lifetime is
short. Never accept a role from a header or request body.

**Signal:** Permission logic duplicated in several handlers with slight differences.

**Why it breaks:** One copy will be wrong, and it will be the copy you did not read.

**Fix:** One `can(user, action, resource)` function, called from every handler. Divergence becomes
impossible rather than merely unlikely.

---

## Multi-tenancy leaks

**Signal:** Queries filtered by user but not by organization/team/workspace. A tenant id taken from
a header or request body. A shared cache key without the tenant in it.

**Why it breaks:** In a B2B app, cross-tenant data exposure is usually the worst possible outcome —
one customer reading another customer's data ends contracts.

**Fix:**
- Derive the tenant from the session, never from the request.
- Put the tenant filter in a place that cannot be forgotten: a base query scope, an ORM global scope,
  a repository layer, or database row-level security.
- Include the tenant id in every cache key, queue message, and file path.
- Test explicitly: create two tenants, authenticate as one, request the other's resources, expect 404.

---

## Client-trusted authorization

**Signal:** Prices, totals, discounts, quantities, plan tiers, or feature flags read from the request
body and used without re-derivation.

```js
await stripe.charges.create({ amount: req.body.amount });   // client sets the price
```

**Why it breaks:** The frontend is not a trust boundary. Anyone can send any body. This is how a
$500 order becomes a $0.50 order.

**Fix:** Send identifiers, derive values server-side. Accept `{ productId, quantity }`, look the price
up in the database, compute the total, and validate quantity bounds. The same applies to
`isPremium`, `discountPercent`, `shippingCost`, and `role`.

---

## Indirect object exposure

**Signal:** Sequential integer primary keys exposed in URLs. Full objects serialized with
`res.json(user)`, `fields = '__all__'`, or `toJSON()` with no allowlist.

**Why it breaks:** Sequential ids make enumeration trivial once an IDOR exists, and leak business
volume. Whole-object serialization ships `passwordHash`, `stripeCustomerId`, `internalNotes`,
`email` of other users, and soft-deleted rows.

**Fix:** Opaque ids (UUIDv4/ULID) for anything user-facing — defense in depth, not a substitute for
ownership checks. Define an explicit response shape per endpoint listing the fields that go out.
Confirm that related/included records are filtered too: an ownership check on the parent does not
authorize the children an ORM eagerly loads alongside it.

---

## Verifying the fix

For each authorization fix, confirm all four:

1. The owner can still perform the action.
2. A different authenticated user gets 404 (or 403 where existence is not sensitive).
3. An unauthenticated caller gets 401.
4. The check cannot be bypassed by moving the id from the path to the body, or by using a related
   endpoint that touches the same object.
