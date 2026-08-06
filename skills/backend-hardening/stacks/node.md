# Node / TypeScript

Express, NestJS, Fastify, Hono, Koa, Next.js route handlers and server actions.

Framework-specific signals and defaults. Use alongside the `references/` files, which hold the
general rules.

## Contents

- What to check first
- Express
- NestJS
- Fastify and Hono
- Next.js
- Prisma, Drizzle, TypeORM, Mongoose
- TypeScript-specific holes
- Grep list

---

## What to check first

1. `package.json` — framework, ORM, validation library, and whether `helmet`/`cors`/rate limiting
   are present at all. Check `"type"`, the Node version in `engines`, and the start script.
2. The entrypoint — middleware order. In Express, order is everything: a guard registered after a
   route does not protect it.
3. Whether async handlers are wrapped (Express 4) — see `errors-and-resilience.md`.

---

## Express

**Middleware order.** Read the file top to bottom and note the line number where each of these is
registered relative to the routes: `helmet()`, `cors()`, body parser, session, auth guard, rate
limiter, error handler. Anything registered after a route does not apply to it. Routers mounted with
`app.use('/x', router)` inherit only the middleware registered before that line.

**The error handler must be last and must take four arguments.** `(err, req, res, next)` — with three
arguments Express treats it as normal middleware and it never runs.

**Async handlers (Express 4).** A rejected promise in `async (req, res) => {}` is not caught. The
request hangs. Fix with `express-async-errors` imported once, a `wrap()` helper, or Express 5.

**Body limits.** `express.json()` defaults to 100kb — good, but confirm nobody raised it globally.
Raise it per route only where needed.

**Trust proxy.** Behind Vercel/Heroku/nginx/Cloudflare, `app.set('trust proxy', 1)` is required for
`req.ip`, `secure` cookies, and rate limiting to work correctly — but never `true`/`'*'`, which lets
a client spoof `X-Forwarded-For` and defeat rate limiting.

**Query parser.** The default `extended` parser turns `?a[b]=c` into an object. Code expecting a
string gets an object — a NoSQL-injection and type-confusion vector. `app.set('query parser', 'simple')`
if you do not need nesting.

**`res.send` after `res.json`.** Two responses on one request throws `ERR_HTTP_HEADERS_SENT`. Usually
a missing `return` before a `res.status(400)`, which also means execution continued into the success
path — check whether that is an auth bypass, not just a warning.

---

## NestJS

**Guards.** `@UseGuards(AuthGuard)` applied per controller is easy to forget on a new controller.
Prefer `APP_GUARD` globally with `@Public()` opting routes out — deny by default.

**ValidationPipe.** Confirm it is registered globally and configured strictly:

```ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,            // strip unknown properties
  forbidNonWhitelisted: true, // 400 on unknown properties
  transform: true,
}));
```

Without `whitelist`, DTO validation does not stop mass assignment — extra fields pass straight
through.

**Serialization.** Without `ClassSerializerInterceptor` and `@Exclude()` on sensitive fields,
returning an entity returns everything including `password`.

**`@Res()`.** Injecting the raw response disables Nest's interceptors and exception filters for that
handler. Check what else was silently turned off.

---

## Fastify and Hono

**Fastify:** schemas are the validation layer — a route without a `schema` is unvalidated. The
response schema also acts as an output allowlist, so adding one is a cheap fix for over-fetching.
Check `@fastify/helmet`, `@fastify/cors`, `@fastify/rate-limit` are registered, and `bodyLimit` is set.

**Hono:** middleware is per-path via `app.use('/api/*', mw)` — confirm the pattern actually covers
the routes, including nested `route()` mounts. `c.req.json()` is unvalidated unless a validator
middleware is attached.

---

## Next.js

**The client/server boundary is the main risk.** A secret imported into a module that is also reached
from a client component ends up in the bundle.

- Any `NEXT_PUBLIC_*` variable is in the browser. See `secrets-and-config.md`.
- `'use server'` functions are **public HTTP endpoints**. They are not protected by the fact that the
  UI only calls them from an authenticated page. Every server action must check auth and
  authorization itself, and validate its arguments — arguments are attacker-controlled.
- Route handlers (`app/api/*/route.ts`) have no automatic auth. Middleware runs before them, but
  confirm the `matcher` actually includes the API paths.
- `middleware.ts` matchers are easy to get wrong. Enumerate routes and check each is covered.
  Do not rely on middleware alone for authorization on sensitive routes — check in the handler too.
- Server components can leak: passing a whole database record as a prop to a client component
  serializes all of it into the HTML payload.
- Check caching on authenticated routes. A personalized response cached at the CDN is served to other
  users. Set `dynamic = 'force-dynamic'` or `no-store` where the response depends on the session.
- `redirect()` and `notFound()` work by throwing — a `try/catch` around them swallows the control
  flow. Check they are not inside a broad catch.

---

## Prisma, Drizzle, TypeORM, Mongoose

**Prisma:** `$queryRawUnsafe` and `$executeRawUnsafe` are injection sinks; `$queryRaw` with a tagged
template is parameterized and safe. `select`/`omit` should be used to keep `password` out of results.
Watch connection pool size on serverless (`connection_limit` in the URL). `findFirst` with only an id
is the classic IDOR shape.

**Drizzle:** `sql.raw()` is the unsafe path; the `sql` tagged template is parameterized. Verify
`where` clauses include the ownership condition.

**TypeORM:** `synchronize: true` must be false in production — it can drop columns. `query()` takes
parameters as a second argument; string-built queries are injectable. `find()` with a
user-supplied object as `where` is a mass-assignment-style risk.

**Mongoose:** `strict` is on by default (good), but `strictQuery` behavior and `findOneAndUpdate`
with a raw body allow operator injection. Coerce ids and values to strings before querying. `select:
false` on the password field, and confirm nothing does `.select('+password')` then returns the doc.

---

## TypeScript-specific holes

**`req.body` is `any`.** TypeScript gives no runtime guarantee. A typed handler signature is
documentation, not validation — the data still needs a schema at runtime.

**`as` casts.** `req.body as CreateUserDto` asserts a lie. Grep for `as ` near request data.

**`strict` off.** Check `tsconfig.json` for `"strict": true`. Without `strictNullChecks`, the
compiler will not tell you a lookup can return `null`, which is where "cannot read property of
undefined" 500s come from.

**Non-null assertions.** `user!.id` on a result that can be null crashes at runtime.

---

## Grep list

```bash
# auth and ownership
rg -n "findById|findUnique|findFirst|findOne" --type ts
rg -n "req\.(params|query|body)\.(id|userId|user_id)" --type ts

# injection
rg -n "queryRawUnsafe|executeRawUnsafe|sql\.raw|\.query\(\s*[\`\"'].*\$\{" --type ts
rg -n "exec\(|execSync\(|child_process" --type ts

# mass assignment
rg -n "\.\.\.req\.body|Object\.assign\(|new \w+\(req\.body" --type ts

# secrets
rg -n "NEXT_PUBLIC_|VITE_|REACT_APP_|EXPO_PUBLIC_" --type ts
rg -n "process\.env\.\w+\s*\|\|\s*['\"]" --type ts     # fallback secrets

# errors
rg -n "catch\s*\([^)]*\)\s*\{\s*\}" --type ts
rg -n "err\.stack|error\.stack" --type ts

# config
rg -n "cors\(\)|origin:\s*['\"]\*|rejectUnauthorized:\s*false" --type ts
rg -n "jwt\.decode\(" --type ts
```
