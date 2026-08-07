# TypeScript and JavaScript ORMs

Prisma, Drizzle, TypeORM, Sequelize, Mongoose, Kysely. Load one ORM file per review; this is the one for
Node and TypeScript projects.

**What to check first, by ORM:**

| ORM | The one thing that matters most |
| :--- | :--- |
| TypeORM | `synchronize: true` — can drop columns on boot |
| Prisma | Missing `@@index` on relation scalar fields |
| Drizzle | The schema file has no `index(` at all |
| Mongoose | `unique: true` is not a validator |
| Sequelize | `limit` combined with a to-many `include` |
| Kysely | The generated types have drifted from the migrations |

Every entry has a **Fine when** line. Read it before reporting.

## Contents

**TypeORM** — `synchronize: true` · `eager: true` and `.limit()` vs `.take()`
**Prisma** — unindexed relation scalars · `relationMode = "prisma"` · `String` on MySQL is
`varchar(191)` · connection pool on serverless · `include` overfetch and N+1
**Drizzle** — no indexes declared · `drizzle-kit push` as the deploy step
**Mongoose** — `autoIndex` and `unique` · `strictQuery` · `populate` N+1 and missing `.lean()`
**Sequelize** — `limit` with a to-many `include`
**Kysely** — type drift

---

## TypeORM: `synchronize: true`

**Signal:** a DataSource or `TypeOrmModule.forRoot({ synchronize: true })` reachable in a non-development
environment. Also `dropSchema: true`.

**Why it breaks:** on every app boot TypeORM diffs entities against the live schema and issues DDL to
make the database match. **A renamed property is seen as "drop old column, add new column"** — the data
in the old column is gone, with no confirmation and no migration file. A removed decorator drops the
column; a removed entity can drop the table.

**Fix:**

```ts
synchronize: false,
migrationsRun: true,
migrations: ['dist/migrations/*.js'],
// typeorm migration:generate -d src/data-source.ts src/migrations/Name
```

**Fine when:** a throwaway local or test database, an in-memory SQLite test harness, or clearly gated —
`synchronize: process.env.NODE_ENV === 'development'` is the safe form.

**Flag the literal `true`**, or a gate reading an env var the deploy config actually sets in production.
Do not flag a correctly gated expression as Critical.

**Corroborating signal:** if `rg --files -g '**/migrations/*.ts'` returns nothing, sync is the only
schema mechanism and the risk is real.

---

## TypeORM: `eager: true` and `.limit()` vs `.take()`

**Signal:** `eager: true` on a relation. Or QueryBuilder pagination written as `.limit(n).offset(m)`
when `.leftJoinAndSelect` is present.

**Why it breaks:** `eager: true` joins that relation into **every** `find()` on the entity, so a list
endpoint silently pulls whole related rows it never uses.

`.limit()` emits raw SQL `LIMIT`, which counts **joined** rows. With a one-to-many join, asking for 20
parents plus their children truncates at 20 raw rows and returns roughly 3 parents. `.take()` wraps the
query so the limit applies to distinct entities.

**Fix:**

```ts
@ManyToOne(() => User) user: User;              // drop eager, load per query
repo.find({ relations: { user: true } })

qb.leftJoinAndSelect('p.tags', 't').take(20).skip(0)
```

**Fine when:** there is no to-many join — `.limit()` is correct and preferred then, because `.take()`
costs an extra query. `eager: true` is fine on a small always-needed to-one relation such as a lookup
table.

---

## Prisma: unindexed relation scalar fields

**Signal:** a model with a relation scalar (`authorId Int` beside
`author User @relation(fields: [authorId], ...)`) and no `@@index([authorId])` or `@unique` covering it.

**Why it breaks:** **Prisma does not emit an index for foreign key columns**, and PostgreSQL and SQLite
do not create one for a FK constraint either. So every `where: { authorId }` lookup and every parent
delete does a sequential scan of the child table. Invisible until a few hundred thousand rows, then list
endpoints and cascading deletes fall off a cliff.

**Fix:**

```prisma
model Post {
  authorId Int
  author   User @relation(fields: [authorId], references: [id])
  @@index([authorId])
}
```

**Fine when:** the provider is `mysql` — InnoDB auto-creates the FK index. Also fine when the field is
already the first column of a `@@unique`/`@@index`, is part of a compound primary key, or the table is a
tiny lookup table.

**Check the `datasource` provider first.**

---

## Prisma: `relationMode = "prisma"`

**Signal:** `relationMode = "prisma"` in the datasource block — typical for PlanetScale, Vitess, and
some Turso setups.

**Why it breaks:** Prisma emits no foreign key constraints at all, so the database creates no index for
the relation either — **even on MySQL**, which would otherwise index it. Prisma's own validator warns
that relation fields "will not benefit from the index usually created by the relational database under
the hood." Referential integrity moves into application code, so any write bypassing Prisma leaves
orphans.

**Fix:** every `@relation(fields: [...])` in the file needs a matching `@@index`. Not optional.

**Fine when:** the platform forbids FK constraints — then `relationMode` itself is correct and required.
**The finding is only the missing `@@index`, never the mode.** Skip the check entirely if the mode is
absent or `"foreignKeys"`.

`npx prisma validate` surfaces the warning.

---

## Prisma: `String` is `varchar(191)` on MySQL

**Signal:** MySQL provider plus bare `String` fields intended to hold long text — `bio String`,
`content String`.

**Why it breaks:** Prisma maps `String` to `varchar(191)` on MySQL. It maps to `text` on
PostgreSQL/SQLite and `nvarchar(1000)` on SQL Server. The 191 comes from the old utf8mb4 index-length
limit, not from the field's meaning. Writing a 200-character bio throws a value-too-long error in
production against data that worked fine on the developer's Postgres.

**Fix:**

```prisma
model Post {
  content String @db.Text
  slug    String @db.VarChar(120)
}
```

**Fine when:** the value is short and bounded — ids, slugs, emails, codes. `varchar` is actually better
there, because `TEXT` columns cannot be fully indexed on MySQL without a prefix length.

Only flag fields whose name or usage implies unbounded text. Not applicable on PostgreSQL.

---

## Prisma: connection pool on serverless

**Signal:** `new PrismaClient()` inside a request handler or serverless function body rather than a
module-scope singleton. Plus a `DATABASE_URL` with no `connection_limit`.

**Why it breaks:** Prisma's default pool is `physical_cpus × 2 + 1` **per PrismaClient instance**, with a
10-second `pool_timeout`. Each concurrent serverless invocation gets its own client and its own pool, so
50 concurrent requests can demand hundreds of Postgres connections and the database starts refusing
them. Symptoms are P2024 pool timeouts and "too many connections" — **under load only**.

**Fix:**

```ts
// lib/prisma.ts — one instance per process
const g = globalThis as any;
export const prisma = g.prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== 'production') g.prisma = prisma;
```

```
DATABASE_URL=...?connection_limit=1&pool_timeout=20&pgbouncer=true
```

**Fine when:** a long-lived single-process server — a container, Fly, Render. One default-sized pool is
exactly right there, and `connection_limit=1` would throttle it.

**Only flag when the deploy target is serverless or edge.** Look for `vercel.json`, `serverless.yml`,
`netlify.toml`, or `export const runtime`.

**Confirm it:** `SELECT count(*), application_name FROM pg_stat_activity GROUP BY 2;`

---

## Prisma: `include` overfetch and N+1

**Signal:** `include:` to pull relations, especially nested `include` inside `include`. `findMany` inside
a `for` loop or `.map()` over a previous result.

**Why it breaks:** `include` returns **every column of both models**, so a list endpoint ships whole rows
— password hashes, large text — over the wire. Without the `relationJoins` preview feature Prisma
resolves each relation level as an additional query, so nested includes multiply round trips.
`findMany` in a loop is a literal N+1.

**Fix:**

```ts
prisma.post.findMany({
  select: { id: true, title: true, author: { select: { id: true, name: true } } }
})
```

```prisma
generator client { previewFeatures = ["relationJoins"] }
```

**Fine when:** a single-record fetch, or the related model is small and you genuinely need all of it. A
`findUnique` inside `Promise.all` over a handful of ids is not a meaningful N+1.

Do not recommend `relationLoadStrategy` on SQLite or MongoDB — it is preview-only and limited to
PostgreSQL, CockroachDB, and MySQL.

---

## Drizzle: no indexes declared

**Signal:** a table whose columns use `.references(() => other.id)` while the table definition has no
third-argument callback declaring `index(...)` — i.e. `pgTable('post', { ... })` with only two
arguments.

**Why it breaks:** `references()` emits a foreign key constraint only. PostgreSQL and SQLite create no
index for it. **Drizzle also never infers indexes from query code**, so a schema written by an AI
typically has zero indexes even for columns filtered on every request.

**Fix:**

```ts
export const post = pgTable('post', {
  id: serial().primaryKey(),
  authorId: integer('author_id').references(() => user.id),
  createdAt: timestamp('created_at'),
}, (t) => [
  index('post_author_idx').on(t.authorId),
  index('post_created_idx').on(t.createdAt.desc()),
]);
```

**Fine when:** the dialect is `mysql-core` (InnoDB auto-indexes FKs), the table is a small lookup table,
or the column is already covered by `primaryKey()` or `unique()` — both create indexes.

**Fast negative test:** a schema file that never imports `index` from `drizzle-orm` has none.

The third argument's return type changed — object form `(t) => ({ ... })` is the older API, array form
`(t) => [ ... ]` the newer one. Both are valid; **do not flag the style.**

---

## Drizzle: `drizzle-kit push` as the deploy step

**Signal:** `drizzle-kit push` in `package.json` scripts with an empty or absent `drizzle/` folder and no
`drizzle-kit generate`.

**Why it breaks:** `push` diffs the TypeScript schema against the live database and applies DDL
immediately, with no versioned artifact and no review. A renamed column looks like drop-plus-add, so the
data goes — and the interactive prompt that would catch it is skipped in CI. There is no rollback and no
way to reproduce the schema on another environment.

**Fix:**

```json
"scripts": {
  "db:generate": "drizzle-kit generate",
  "db:migrate": "drizzle-kit migrate"
}
```

Keep `drizzle/*.sql` and `drizzle/meta` committed.

**Fine when:** local prototyping, or disposable branch/preview databases. Drizzle documents teams using
`push` in production deliberately.

**Only flag it when it is the sole mechanism** — no committed migration files — **and** it runs against a
database holding real data.

---

## Mongoose: `autoIndex` and `unique: true`

**Signal:** fields marked `unique: true` or `index: true`, with `autoIndex` never set to false, and
`mongoose.connect` called in a serverless handler.

**Why it breaks:** `autoIndex` defaults to true, so every process start calls `createIndexes()` — on a
large collection that is a background build that can stall the deploy.

Worse: **`unique: true` is not a validator.** It only asks Mongo for a unique index. If `autoIndex` is
false in production, or the build failed on existing duplicates, duplicate documents are accepted and
the only error you get is a raw `E11000` — or none at all.

**Fix:**

```js
mongoose.connect(uri, { autoIndex: false });
await User.syncIndexes();                    // deliberately, once
// and handle it: err.code === 11000
```

**Fine when:** development or a small collection. The finding is warranted when `autoIndex` is disabled,
the app runs serverless (where index builds race), or no `syncIndexes()`/migration step exists.

Also watch for a field declared both `index: true` and in `schema.index()` — Mongoose 6+ logs that as a
duplicate-index warning.

---

## Mongoose: `strictQuery`

**Signal:** `mongoose.set('strictQuery', ...)` absent, combined with `Model.find(req.query)` or filters
built from user input. Also `strict: false` in a schema.

**Why it breaks:** `strictQuery` defaulted to `true` through Mongoose 6 and flips to **`false` in
Mongoose 7+**. With `false`, a filter key not in the schema is passed to MongoDB verbatim instead of
being stripped — so a typo like `{ userid: id }` matches nothing where it used to match everything, and
vice versa on upgrade. `strict: false` on the schema lets arbitrary request fields be written into
documents.

**Fix:**

```js
mongoose.set('strictQuery', true);

const filter = { userId: req.user.id, status: req.query.status };   // never spread raw input
Model.find(filter);
```

**Fine when:** `strictQuery: false` is deliberate, for querying fields that legitimately exist in
documents but not in the schema — schemaless or legacy collections.

**The real finding is the unfiltered spread of request data into a query**, which is a bug at either
setting. Check the Mongoose version in `package.json` to know which default applies.

---

## Mongoose: `populate` N+1 and missing `.lean()`

**Signal:** `.populate(...)` inside a `for` loop or `.map()`. `.populate()` chains three or more deep.
Read-only route handlers that never call `.lean()`.

**Why it breaks:** a single `populate()` costs one extra query per path, batched with `$in` — that is
fine. Calling it **per document** turns one request into N queries, and deep chains multiply.

Separately, without `.lean()` Mongoose hydrates every result into a full document with change tracking
and virtuals. Mongoose's own docs measure lean results at roughly 3× smaller in memory (~55 bytes vs
~180 in their example), and hydration dominates CPU on large result sets.

**Fix:**

```js
const posts = await Post.find(q).populate('author', 'name avatar').lean();
// or one $lookup in an aggregate() for deep chains
```

**Fine when:** a single top-level `populate()` on a list query — that is normal.

**`.lean()` must NOT be added** where the result is mutated and `save()`d, or where virtuals, getters, or
`toJSON` transforms are relied on. That is a silent behaviour change — only recommend it for read paths.

---

## Sequelize: `limit` with a to-many `include`

**Signal:** `findAll`/`findAndCountAll` combining `limit` with an `include` of a hasMany or
belongsToMany association, without `separate: true`. Or `subQuery: false` alongside `limit`.

**Why it breaks:** with a to-many include the join produces one row per child, so Sequelize wraps the
query in a subquery to make `LIMIT` count parents. **Setting `subQuery: false` removes that wrapper**, so
`LIMIT` then counts joined rows and the endpoint returns the wrong number of parents. Left as a join,
the parent row is duplicated once per child and de-duplicated in JS — heavy payloads on wide parent
tables. `findAndCountAll` also reports the duplicated count.

**Fix:**

```js
Post.findAll({
  limit: 20,
  include: [{ model: Comment, separate: true, limit: 5, order: [['createdAt','DESC']] }],
});

Post.findAndCountAll({ include: [Comment], distinct: true });
```

**Fine when:** the include is belongsTo or hasOne — one row per parent — where `subQuery: false` is
legitimate and often necessary to filter on a joined column.

**Neither strategy is universally better**, and Sequelize's docs say so. A join is cheaper when each
parent has very few children; `separate: true` wins when children are numerous or you need a per-parent
`order`/`limit`.

---

## Kysely: type drift

**Signal:** a hand-written `interface Database` with no `kysely-codegen` (or `prisma-kysely`) script, or
a generated types file older than the newest migration.

**Why it breaks:** Kysely's type safety is only as good as that interface — **nothing at runtime checks
it against the real database.** When a migration adds a `NOT NULL` column and the interface is not
regenerated, the compiler happily approves inserts the database rejects, and `selectAll()` returns
columns typed as present that are not. It fails only in production.

**Fix:**

```json
"scripts": { "db:types": "kysely-codegen --out-file src/db/types.ts" }
```

Run it in CI after migrations and fail the build on a diff. Prefer explicit columns over `selectAll()`.

**Fine when:** the schema is small and stable — a hand-written interface is a valid choice and not itself
a defect.

**The finding is drift.** Only raise it when migrations are demonstrably newer than the types, or there
is no regeneration step in CI. Compare
`git log -1 --format=%ci -- src/db/types.ts` against the newest migration.

Kysely never creates indexes implicitly — check migrations for `createIndex(`.

---

## Sources

Prisma: [indexes](https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes),
[relation-mode](https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/relation-mode),
[connection-pool](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/connection-pool),
[relation-queries](https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries).
TypeORM: [data-source-options](https://typeorm.io/data-source-options).
Drizzle: [indexes-constraints](https://orm.drizzle.team/docs/indexes-constraints),
[migrations](https://orm.drizzle.team/docs/migrations).
Mongoose: [guide](https://mongoosejs.com/docs/guide.html),
[lean](https://mongoosejs.com/docs/tutorials/lean.html).
Sequelize: [select-in-depth](https://sequelize.org/docs/v7/querying/select-in-depth/).
