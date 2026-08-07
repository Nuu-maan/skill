# Ruby, JVM, and .NET ORMs

ActiveRecord, Hibernate/JPA, and EF Core. Load one ORM file per review; this is the one for Rails,
Spring, and ASP.NET projects.

**What to check first:**

| Stack | The one thing that matters most |
| :--- | :--- |
| Rails | `add_index` with no `algorithm: :concurrently` |
| Hibernate/JPA | `ddl-auto: update` outside a test profile |
| Hibernate/JPA | `@ManyToOne` defaults to **EAGER** — the opposite of collections |
| EF Core | Unconfigured `string` → `nvarchar(max)`, unindexable on SQL Server |

Every entry has a **Fine when** line. Read it before reporting.

## Contents

**ActiveRecord** — index builds and `disable_ddl_transaction!` · `strong_migrations` ·
unvalidated constraints · `includes` vs `preload` vs `eager_load` · `dependent: :destroy` ·
counter caches
**Hibernate / JPA** — `@ManyToOne` is EAGER by default · `ddl-auto` · collection N+1 and
`open-in-view` · `join fetch` with pagination · `@Enumerated` ORDINAL
**EF Core** — unconfigured column types · migration renames · `AsNoTracking` · lazy proxies and
cartesian `Include`s · client evaluation

---

## Rails: index builds and `disable_ddl_transaction!`

**Signal:** a file under `db/migrate/` with `add_index`, `add_reference`, or `t.references ... index:
true` and no `algorithm: :concurrently` — or with `algorithm: :concurrently` but **no
`disable_ddl_transaction!`** at the top of the class.

**Why it breaks:** on PostgreSQL a plain `CREATE INDEX` takes a SHARE lock and blocks every write on the
table for the whole build — minutes on a multi-million-row table, which reads as a total outage.
Conversely, `algorithm: :concurrently` inside Rails' default DDL transaction raises
`CREATE INDEX CONCURRENTLY cannot run inside a transaction block`, and a half-applied concurrent build
leaves an **INVALID** index that is never used and must be dropped by hand.

**Fix:**

```ruby
class AddIndex < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!

  def change
    add_index :users, :email, algorithm: :concurrently
    add_reference :users, :city, index: { algorithm: :concurrently }
  end
end
```

**Fine when:** the index is created inside the same `create_table` block — the table is brand new and
empty — or the migration only ever ran against a development database. Also fine on MySQL/MariaDB, which
do online DDL by default. **Do not flag `create_table do |t| t.index ... end`.**

`disable_ddl_transaction!` is a class-body statement, so check the whole file, not the method.

---

## Rails: `strong_migrations`

**Signal:** no `strong_migrations` in the `Gemfile`. Or migrations peppered with `safety_assured { ... }`
blocks.

**Why it breaks:** strong_migrations is the de-facto guard that fails the migration **at development
time** for the whole family of locking operations — removing or renaming a column, changing a column
type, adding a FK or check constraint without `validate: false`, backfilling in the same transaction as
an ALTER. Without it, nothing stops an AI-written migration from shipping a table rewrite.
`safety_assured` is the opt-out, so a repo full of it has the guard installed and disabled.

**Fix:**

```ruby
gem "strong_migrations"
# rails generate strong_migrations:install

StrongMigrations.start_after = 20240101000000
StrongMigrations.target_version = 16      # your Postgres major
```

**Fine when:** a single `safety_assured` with a comment explaining why — a table known to be empty, a
documented dual-write sequence already completed. That is correct usage, not a smell.

Absence of the gem is only worth reporting for an app with real production traffic. A greenfield repo
does not need it yet, and `start_after` legitimately excludes historical migrations.

---

## Rails: unvalidated constraints

**Signal:** `change_column_null :t, :c, false`, `add_foreign_key`, or `add_check_constraint` with no
`validate: false`, and no second migration calling `validate_foreign_key` / `validate_check_constraint`.

**Why it breaks:** each makes Postgres scan and validate every existing row while holding a lock that
blocks reads and writes — **both tables**, for a foreign key. Splitting into add-unvalidated then
validate-later takes only a brief lock in the first step, and the validation pass takes a weaker lock
that lets traffic through.

**Fix:**

```ruby
# migration 1
add_check_constraint :users, "email IS NOT NULL", name: "users_email_null", validate: false

# migration 2
validate_check_constraint :users, name: "users_email_null"
change_column_null :users, :email, false
remove_check_constraint :users, name: "users_email_null"
```

**Fine when:** the constraint is declared inline in `create_table` on a new table, the table is under
~100k rows, or the database is MySQL — which blocks writes rather than reads and has no
`validate: false`.

The two-step dance is unnecessary on an empty table and only adds noise.

---

## Rails: `includes` vs `preload` vs `eager_load`

**Signal:** `.includes(:assoc).where("assocs.col = ?", x)` with no `.references(:assoc)`. Or `.joins(:assoc)`
followed by code reading `record.assoc` in a loop.

**Why it breaks:** `preload` is always separate queries, `eager_load` is always one LEFT OUTER JOIN, and
`includes` **guesses** between them — it only picks the JOIN if you tell it, via `references` or a hash
condition. A raw SQL-string condition without `references` raises or silently misses. And `joins` alone
loads no association data at all, so the loop that reads it is a textbook N+1 that no schema change will
fix.

**Fix:**

```ruby
Author.joins(:books).where(books: { out_of_print: true })        # filter only
Author.eager_load(:books).where(books: { out_of_print: true })   # filter AND read: one JOIN
Author.preload(:books)                                            # read only, no cartesian blowup
Author.includes(:books).where("books.out_of_print = true").references(:books)
```

**Fine when:** `includes` carries a hash condition (`where(books: {...})`) — Rails infers the
`references` there.

Prefer `preload` only when the parent row is narrow; if the parent has a large text or blob column,
`eager_load`'s row duplication may still be cheaper than an extra round trip.

**Also worth flagging:** an `.includes` in the controller for an association the view never touches. That
is dead weight.

---

## Rails: `dependent: :destroy`

**Signal:** `has_many :things, dependent: :destroy` where children number in the thousands. And/or
`add_foreign_key` with no `on_delete:` clause.

**Why it breaks:** `:destroy` instantiates **every child row** into a Ruby object and fires its callbacks
one DELETE at a time. Deleting a user with 500k events is 500k queries inside one transaction — it times
out and holds locks the whole while.

**Fix:**

```ruby
has_many :events, dependent: :delete_all           # no callbacks, no grandchildren
add_foreign_key :events, :users, on_delete: :cascade   # or push it to the database
```

For large tables, delete in batches from a background job instead.

**Fine when:** the child model has destroy callbacks, `has_many :through` cleanup, Active Storage
attachments, or its own dependent children. **Downgrading `:destroy` there is a data-integrity bug.**
Only flag it when the child is a leaf with no callbacks.

`on_delete: :cascade` bypasses all Rails callbacks, including Active Storage purge.

---

## Rails: counter caches

**Signal:** `post.comments.count` / `.size` inside a loop, with no `counter_cache: true` on the child's
`belongs_to`. Or a migration adding `comments_count` with no backfill step.

**Why it breaks:** `.count` in a loop over 50 rows is 50 `COUNT(*)` queries. **Adding the counter column
the naive way is worse:** the column defaults to 0, live traffic starts incrementing from 0, and every
existing record shows a wrong count forever because the backfill and the increments race.

**Fix — four steps, in order:**

```ruby
# 1. column
add_column :posts, :comments_count, :integer, default: 0, null: false

# 2. Rails 7.1+: declare it inactive while backfilling
belongs_to :post, counter_cache: { active: false }

# 3. backfill
Post.find_each { |p| Post.reset_counters(p.id, :comments) }

# 4. deploy again with plain counter_cache: true
```

**Fine when:** a single `.count` outside a loop — often better than a stale cache.

Counter caches are **wrong** when the count must be filtered (`comments.approved.count`). On Rails < 7.1
the safe backfill is a maintenance window or a dual-write; do not recommend `{ active: false }` there.

A `_count` column with no matching `counter_cache` declaration is a dead column that silently drifts.

---

## JPA: `@ManyToOne` is EAGER by default

**Signal:** `@ManyToOne` or `@OneToOne` with no `fetch` attribute.

**Why it breaks:** JPA's default for `@ManyToOne` and `@OneToOne` is **`FetchType.EAGER`** — the opposite
of the collection defaults, which is why it surprises everyone. Every load of the child silently joins or
issues an extra SELECT for the parent, **and the parents chain**: loading one `OrderLine` can drag in
Order, Customer, and Address. In a `findAll()` this multiplies into hundreds of queries nobody wrote.

**Fix:**

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id")
private User user;

@OneToOne(fetch = FetchType.LAZY, optional = false)
private Profile profile;
```

**Fine when:** a small always-needed lookup (a `Currency` reference) where one extra join is cheaper than
a second query.

**Important limitation:** a lazy `@OneToOne` on the non-owning (`mappedBy`) side, or an owning side with
`optional = true`, **cannot actually be made lazy** without bytecode enhancement — Hibernate must query
to know whether to store null or a proxy. `LAZY` there is a no-op, not a fix. Say so rather than
recommending it.

Explicitly `EAGER` collections (`@OneToMany(fetch = EAGER)`) are worse and always worth flagging.

---

## JPA: `ddl-auto`

**Signal:** `spring.jpa.hibernate.ddl-auto: update` (or `create`, `create-drop`), or
`hibernate.hbm2ddl.auto` set the same, in a profile that is not `test`.

**Why it breaks:** `update` diffs entities against the live database and issues ALTERs at startup. It
never drops or narrows anything, so removed fields linger as orphaned `NOT NULL` columns, and every
startup can add an unreviewed column or index. `create` and `create-drop` **drop the whole schema.**
There is no ordering, no review, and no rollback.

**Fix:**

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

Schema changes go through Flyway or Liquibase: `src/main/resources/db/migration/V3__add_index.sql`.

**Fine when:** `create-drop` in `application-test.yml`, `src/test/resources/`, or a Testcontainers setup.
**Do not flag those.** `update` is also defensible in a local-dev-only profile.

`validate` is the right production value **only when a real migration tool owns the schema** — with no
migration tool at all, `validate` will simply fail to start.

Spring Boot defaults to `create-drop` for embedded databases (H2, HSQLDB, Derby) and `none` otherwise,
unless Flyway or Liquibase is on the classpath.

---

## JPA: collection N+1 and `open-in-view`

**Signal:** a `@OneToMany` collection read inside a loop over repository results, with no `join fetch`,
`@EntityGraph`, or `@BatchSize` anywhere. Plus `spring.jpa.open-in-view` unset.

**Why it breaks:** lazy collections default to one SELECT per parent on first access — 1 query plus N.
And **`open-in-view` defaults to true** in Spring Boot, which keeps the session open through JSON
rendering, so the N+1 fires during serialization where no profiler in the service layer will see it —
while holding a database connection for the whole request.

**Fix:**

```java
@Query("select distinct o from Order o join fetch o.lines where o.status = :s")
List<Order> findWithLines(@Param("s") Status s);

@EntityGraph(attributePaths = "lines")
List<Order> findByStatus(Status s);
```

```yaml
spring.jpa.properties.hibernate.default_batch_fetch_size: 50   # turns N queries into N/50
spring.jpa.open-in-view: false
```

**Fine when:** the collection is accessed on a single entity. A `findById` that touches one collection
issues 2 queries and is fine — it only matters when accessed per element of a multi-row result.

**`open-in-view: false` is the right default but will surface `LazyInitializationException`** in code
that relied on it. That is a real and correct refactor, not a one-line flip — say so.

**Confirm at runtime:** `spring.jpa.properties.hibernate.generate_statistics=true` plus
`logging.level.org.hibernate.SQL=DEBUG`.

---

## JPA: `join fetch` with pagination

**Signal:** a repository method combining `join fetch` on a collection with `Pageable` or
`setMaxResults`. Or one JPQL query with two `join fetch` clauses onto `List<...>` fields.

**Why it breaks:** Hibernate cannot apply SQL `LIMIT`/`OFFSET` to a collection-fetch join without slicing
a parent's children in half, so it **fetches the entire result set and paginates in memory** — logging
`HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!` and risking OOM.
Two collection fetches at once produce a cartesian product; if both are `List` (a "bag"), Hibernate
refuses outright with `MultipleBagFetchException`.

**Fix:** paginate IDs first, then fetch.

```java
@Query("select o.id from Order o where o.status = :s")
Page<Long> findIds(@Param("s") Status s, Pageable p);

@Query("select distinct o from Order o join fetch o.lines where o.id in :ids")
List<Order> fetchLines(@Param("ids") List<Long> ids);
```

**Fine when:** the `join fetch` targets a `@ManyToOne`/`@OneToOne`. The row count does not multiply, so
pagination is completely safe and no warning fires.

Switching `List` to `Set` to dodge `MultipleBagFetchException` **still creates a cartesian product** in
the SQL — it only works when both collections are genuinely small.

Hibernate 6 narrowed HHH000104 to fire only when the query really contains a fetch join.

---

## JPA: `@Enumerated` ORDINAL

**Signal:** an enum field annotated `@Enumerated` with no argument, or with no annotation at all. Also
`String` fields with no `@Column(length = ...)`.

**Why it breaks:** `@Enumerated`'s default is **`EnumType.ORDINAL`**, which stores the enum's positional
index. Inserting or reordering a constant silently reinterprets every existing row — `ACTIVE` becomes
`DELETED` — with no error and no way to recover from the data alone.

Unannotated `String` becomes `varchar(255)` regardless of content, and unannotated `BigDecimal` becomes
`decimal(19,2)`, truncating anything finer.

**Fix:**

```java
@Enumerated(EnumType.STRING)
@Column(length = 32, nullable = false)
private Status status;

@Column(precision = 19, scale = 4)
private BigDecimal amount;
```

**Fine when:** the enum is documented as append-only and the ordering is pinned by a test. `varchar(255)`
is fine for genuinely short free text.

**If the schema already stores ordinals in production**, switching to `STRING` requires a data migration.
Report it as "freeze this enum's order and pin it with a test," not "change the annotation."

---

## EF Core: unconfigured column types

**Signal:** `public string Name { get; set; }` with no `[MaxLength]`/`HasMaxLength`, and
`public decimal Price { get; set; }` with no `[Precision]`/`HasPrecision`.

**Why it breaks:** the SQL Server provider maps an unconfigured `string` to **`nvarchar(max)`** — which
cannot be part of an index key at all, is stored off-row, and kills any query filtering on it. An
unconfigured `decimal` gets `decimal(18,2)`, so a tax rate or a crypto amount is **silently truncated to
two places**; EF logs `DecimalTypeDefaultWarning` but nothing fails.

**Fix:**

```csharp
modelBuilder.Entity<Product>(e => {
    e.Property(p => p.Name).HasMaxLength(200);
    e.Property(p => p.Sku).HasMaxLength(64).IsUnicode(false);
    e.Property(p => p.Price).HasPrecision(18, 6);
});
```

**Fine when:** `nvarchar(max)` for genuine long text — descriptions, JSON blobs, markdown — that is never
indexed or compared. `decimal(18,2)` is correct for plain currency.

**Only flag columns that are filtered, joined, sorted on, or hold more than 2 decimal places.**

Npgsql maps unconfigured `string` to `text` (indexable, so lower severity) and `decimal` to `numeric`
with no precision, so there is no truncation there.

**Fastest detection:** grep the migration output, which is unambiguous —
`rg -n 'nvarchar\(max\)|decimal\(18,2\)' Migrations/*.cs`.

---

## EF Core: migration renames

**Signal:** in `Migrations/*.cs`, a `DropColumn` + `AddColumn` pair for what is obviously a rename. A
`DropColumn` shipped in the same migration as the code that stops using it. `CreateIndex` on a Postgres
project with no `suppressTransaction`.

**Why it breaks:** EF Core's scaffolder cannot see intent — renaming a C# property emits drop-then-add,
**which destroys the column's data on apply.** Dropping a column in the same deploy as the code change
means old app instances query a column that no longer exists during the rollout. On Postgres, EF wraps
each migration in a transaction, so `CREATE INDEX CONCURRENTLY` fails outright unless suppressed.

**Fix:**

```csharp
migrationBuilder.RenameColumn(name: "Url", table: "Blogs", newName: "Address");

migrationBuilder.Sql(
    "CREATE INDEX CONCURRENTLY IX_Posts_BlogId ON \"Posts\" (\"BlogId\");",
    suppressTransaction: true);
```

**Fine when:** the type changed incompatibly and the data is meant to be discarded, or the migration has
only ever run against a development database.

A suppressed transaction means a **failed migration leaves the index behind and is not rolled back** —
that trade is only worth it on a large live table.

Read `dotnet ef migrations script --idempotent` output rather than the C# where you can.

---

## EF Core: `AsNoTracking`

**Signal:** read-only query paths — API GETs, report endpoints, `.ToListAsync()` feeding a DTO — with no
`AsNoTracking()` and no `QueryTrackingBehavior.NoTracking` in `AddDbContext`.

**Why it breaks:** by default EF Core snapshots every returned entity into the change tracker to compute
updates later. On a 10,000-row read that is roughly 2× the memory and a measurable per-row CPU cost for
work that is thrown away, and it makes the DbContext progressively slower the longer it lives.

**Fix:**

```csharp
var rows = await db.Orders.AsNoTracking().Where(o => o.Status == Status.Open).ToListAsync();
// or globally: options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
```

**Never add it** to a query whose results are then modified and saved — `SaveChanges` will silently do
nothing.

A projection to a DTO via `.Select(...)` is **already untracked**, so `AsNoTracking` there is redundant,
not a fix. If the same entity appears many times in one no-tracking result and reference identity
matters, use `AsNoTrackingWithIdentityResolution`.

---

## EF Core: lazy proxies and cartesian `Include`s

**Signal:** `UseLazyLoadingProxies()` plus `public virtual ICollection<T>` navigations. And/or two or more
**sibling** `.Include()` calls on collections in one query with no `.AsSplitQuery()`.

**Why it breaks:** lazy-loading proxies turn every property read into a possible round trip — an N+1 that
is invisible in the query code because there is no query.

Two sibling collection includes produce a **cross product**: a blog with 10 posts and 10 contributors
returns 100 rows, and it multiplies with each extra `Include`. EF Core logs a
`MultipleCollectionIncludeWarning` when it detects this.

**Fix:**

```csharp
var blogs = await db.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Contributors)
    .AsSplitQuery()
    .ToListAsync();
```

**Fine when:** the includes are *nested* — `Include(b => b.Posts).ThenInclude(p => p.Comments)` — which
produces no cross product. **Do not flag `ThenInclude`.** One-to-one navigations are always joined and
never split.

Split queries cost an extra round trip each and give up single-query consistency. Before EF Core 10 they
also need a fully unique `OrderBy` when combined with `Skip`/`Take`, or they can return wrong rows.

---

## EF Core: client evaluation

**Signal:** `.ToList()` / `.AsEnumerable()` appearing **before** a `.Where()`, `.OrderBy()`, or
`.Count()` in a LINQ chain. Or a `.Where()` whose predicate calls a local helper method.

**Why it breaks:** once the chain hits `ToList`, everything after it runs in memory — the app pulls the
entire table over the wire and filters it in C#. EF Core 3.0 made untranslatable predicates an explicit
runtime failure rather than a silent degradation, but an explicit `.ToList()` mid-chain still compiles
and still fetches everything.

**Fix:**

```csharp
// wrong: fetches every order
var open = db.Orders.ToList().Where(o => o.Status == Status.Open);

// right: filter in SQL
var open = await db.Orders.Where(o => o.Status == Status.Open).ToListAsync();
```

Helper methods cannot translate — inline the predicate or move it into an `EF.Functions` form.

**Fine when:** the result set is already small (a lookup table, an already-filtered page), or the
operation has no SQL translation.

**EF Core permits client evaluation in the top-level projection by design** —
`Select(b => new { Url = Standardize(b.Url) })` is supported and correct. Only flag a materialisation
that precedes a filter or an aggregate over an unbounded set.

---

## Sources

Rails: [strong_migrations](https://github.com/ankane/strong_migrations),
[active record querying](https://guides.rubyonrails.org/active_record_querying.html),
[association basics](https://guides.rubyonrails.org/association_basics.html).
Spring / Hibernate: [data initialization](https://docs.spring.io/spring-boot/how-to/data-initialization.html),
[HHH000104](https://vladmihalcea.com/fix-hibernate-hhh000104-entity-fetch-pagination-warning-message/),
[MultipleBagFetchException](https://vladmihalcea.com/hibernate-multiplebagfetchexception/).
EF Core: [entity properties](https://learn.microsoft.com/en-us/ef/core/modeling/entity-properties),
[single vs split queries](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries),
[client evaluation](https://learn.microsoft.com/en-us/ef/core/querying/client-eval),
[managing migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/managing).
