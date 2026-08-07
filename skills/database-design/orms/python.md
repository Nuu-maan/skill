# Python ORMs

Django ORM with its migration system, and SQLAlchemy with Alembic. Load one ORM file per review; this is
the one for Python projects.

**What to check first:**

| Stack | The one thing that matters most |
| :--- | :--- |
| Django + DRF | `queryset = Model.objects.all()` with no `select_related` |
| Django migrations | A `RemoveField` + `AddField` pair that was meant to be a rename |
| SQLAlchemy | `relationship()` with no `lazy=` — lazy select is the default |
| Alembic | `compare_server_default` is off by default |

Every entry has a **Fine when** line. Read it before reporting.

## Contents

**Django models** — `CASCADE` fan-out · `on_delete` on attribution fields · redundant `db_index=True`
**Django queries** — N+1 in serializers and templates · breaking the prefetch cache ·
`only()`/`defer()` backfiring
**Django migrations** — what `makemigrations` cannot see · renames become drop+add · index builds lock
**SQLAlchemy** — lazy loading · `joinedload` on collections · session lifecycle
**Alembic** — autogenerate blind spots · unnamed constraints

---

## Django: `CASCADE` fan-out

**Signal:** `on_delete=models.CASCADE` on a ForeignKey whose parent has many children — `User` → events,
logs, audit rows.

**Why it breaks:** Django's docs state plainly that "on_delete doesn't create an SQL constraint in the
database" — CASCADE is emulated in Python. Deleting one parent makes Django `SELECT` **every descendant
row into memory** across the whole relation graph, then issue DELETEs, firing `pre_delete`/`post_delete`
per object. Deleting a user with 500k events can take minutes and OOM the worker. The database would
have done it in one statement.

**Fix:**

```python
# high-volume child tables where signals are not needed
user = models.ForeignKey(User, on_delete=models.DO_NOTHING, db_constraint=False)
```

Then enforce cleanup with a real constraint in a `RunSQL` migration:

```sql
ALTER TABLE events ADD CONSTRAINT ... FOREIGN KEY (user_id)
  REFERENCES users(id) ON DELETE CASCADE;
```

**Fine when:** the children are genuinely owned and low-cardinality — an Address on a Profile, an
OrderLine on an Order. That is the correct default.

**Only a problem for fan-out into log, event, and audit tables.** And Python-side CASCADE is what makes
`post_delete` signals and `FileField` cleanup work — do not convert to database-level cascade if the
project relies on those.

---

## Django: `on_delete` on attribution fields

**Signal:** `on_delete=models.CASCADE` on a nullable attribution FK — `created_by`, `assigned_to`,
`approved_by`, `reviewer`. Also `SET_NULL` without `null=True`, or `SET_DEFAULT` without a `default=`.

**Why it breaks:** a `created_by` FK set to CASCADE means **deleting a staff account silently deletes
every record they ever created.** `PROTECT` or `RESTRICT` would have raised `ProtectedError` instead.

`SET_NULL` without `null=True` raises at check time, so it fails loudly — the dangerous variant is the
one that passes checks.

**Fix:**

```python
created_by = models.ForeignKey(
    User, null=True, blank=True,
    on_delete=models.SET_NULL,        # keep the record, lose the attribution
    related_name="created_widgets",
)

currency = models.ForeignKey(Currency, on_delete=models.PROTECT)   # lookup rows
```

**Fine when:** the child genuinely should not outlive the parent — a user's own drafts under a hard-delete
erasure policy.

`PROTECT` is wrong where the app expects deletes to succeed; it turns a delete into an unhandled 500
unless `ProtectedError` is caught. `RESTRICT` (Django 3.0+) differs only in allowing the delete when the
same operation cascades the referrer away.

**Find it:** flag field names matching
`created_by|updated_by|author|owner|assigned|approved|reviewer|actor` that use CASCADE.

---

## Django: redundant `db_index=True`

**Signal:** `db_index=True` on a `CharField`/`TextField`, alongside `unique=True` on the same field, or
on a `ForeignKey`.

**Why it breaks:** on **PostgreSQL**, `db_index=True` on a char or text column creates **two** indexes —
the btree plus a second `..._like` index using `varchar_pattern_ops`. Double the write cost and disk for
a LIKE-prefix capability most apps never use. And `unique=True` already creates a unique index, while
`ForeignKey` is indexed by default, so both make the extra index pure overhead.

**Fix:**

```python
author = models.ForeignKey(User, on_delete=models.PROTECT)   # already indexed
slug   = models.SlugField(unique=True)                        # already indexed

class Meta:
    indexes = [models.Index(fields=["status", "-created_at"])]   # control the shape
```

**Fine when:** the codebase uses `__startswith`, `__contains`, or `__istartswith` on that column — then
the `_like` index is genuinely needed. **Grep for those before recommending removal.**

MySQL and SQLite create no second index, so this entry is Postgres-only. `db_index=True` on a ForeignKey
is harmless noise rather than a defect — Django ignores the duplicate.

---

## Django: N+1 in serializers and templates

**Signal:** a DRF serializer with a nested serializer or `SerializerMethodField` walking a relation, and
a `ModelViewSet` whose `queryset = Model.objects.all()` has no `select_related`/`prefetch_related`. Same
shape in a template: `{% for x in qs %}{{ x.author.name }}`.

**Why it breaks:** one SELECT per row per relation. A 50-item list page with a nested author and a tags
list becomes 101 queries. Invisible in development with 5 rows of seed data; takes the endpoint from
20ms to 2s in production.

**Fix:**

```python
class WidgetViewSet(ModelViewSet):
    queryset = (Widget.objects
                .select_related("author", "category")            # FK / O2O -> JOIN
                .prefetch_related("tags", "comments__author"))   # M2M / reverse FK
```

**Fine when:** the endpoint is a detail view returning one object, or the relation is already prefetched
by a shared `get_queryset()` on a base class — **check the MRO before flagging.**

Over-applying `select_related` is its own bug: chaining it across several nullable FKs produces a wide
LEFT OUTER JOIN that can be slower than two queries. It only works on ForeignKey and OneToOneField;
using it on M2M or a reverse FK raises.

---

## Django: breaking the prefetch cache

**Signal:** `.prefetch_related("toppings")` followed anywhere by `obj.toppings.filter(...)`,
`.count()`, `.exclude(...)`, or `.order_by()` on the related manager.

**Why it breaks:** Django's docs are explicit — "pizza.toppings.filter() is a new and different query.
The prefetched cache can't help here; in fact it hurts performance, since you have done a database query
that you haven't used." **You pay for the prefetch and still get the N+1.**

**Fix:**

```python
from django.db.models import Prefetch

qs = Pizza.objects.prefetch_related(
    Prefetch("toppings",
             queryset=Topping.objects.filter(spicy=True),
             to_attr="spicy_toppings"))
# read pizza.spicy_toppings — a plain list, no query
# and use len(pizza.toppings.all()) instead of .count()
```

**Fine when:** the filtered access happens on a single object outside the prefetched loop, or the
prefetch serves a different consumer of the same queryset.

`.all()` on a prefetched manager **does** hit the cache — only query-altering methods break it.
`add()`/`create()`/`remove()`/`clear()`/`set()` also clear the cache, which is correct behaviour.

---

## Django: `only()` and `defer()` backfiring

**Signal:** `.only("id", "name")` or `.defer("body")` on a queryset whose results are later serialized,
rendered in a template, or passed to code you do not control.

**Why it breaks:** accessing a deferred field on a loaded instance triggers a fresh SELECT for that one
field on that one row. `.only('id')` over 100 objects that a serializer then reads 3 fields from is
**301 queries** — strictly worse than no optimisation. `only()` combined with `select_related()` also
silently defers the related model's fields unless you name them.

**Fix:**

```python
qs = (Article.objects
      .select_related("author")
      .only("id", "title", "created_at", "author__id", "author__name"))

qs = Article.objects.defer("raw_html_body")     # safer default
```

**Fine when:** excluding a genuinely large TEXT/JSON/binary column from a list query — article bodies,
blobs, embeddings. That is the one case where it earns its risk.

Almost always wrong on a small model with only scalar columns; the row is one page read either way.
`.values()`/`.values_list()` is the safer tool when you truly want narrow rows, because it cannot lazily
re-fetch.

**Find it:** for each hit, diff the `only()` list against the consumer's field list. Any field in the
consumer but not in `only()` is a per-row query.

---

## Django: what `makemigrations` cannot see

**Signal:** any migration folder. The blind spot is structural — **`makemigrations` diffs your models
against the state reconstructed from existing migration files, never against the live database.**

**Why it breaks:** if anyone hand-edited a migration, applied SQL manually, ran `--fake`, or deleted a
migration file, `makemigrations` reports "No changes detected" while the real schema diverges. It
surfaces months later as an `OperationalError` in production only.

**Fix:**

```bash
python manage.py makemigrations --check --dry-run   # models == migration graph
python manage.py sqlmigrate <app> <migration>       # inspect the emitted DDL
```

Then compare against the database's own view:

```sql
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = '<table>';
```

**Fine when:** `RunSQL` carries a correct `state_operations=[...]` argument — that is the sanctioned way
to keep model state and database state aligned. **Check for that kwarg before flagging.** `--fake` is
legitimate during a squash or when first adopting migrations on an existing database.

**Every `RunSQL` without `state_operations` is a place where the two can silently diverge**, because
Django does not parse the SQL.

---

## Django: renames become drop + add

**Signal:** a migration containing `migrations.RenameField`/`RenameModel` — or, worse, an adjacent
`RemoveField` + `AddField` pair on the same table with similar names.

**Why it breaks:** Django **cannot detect renames**; `makemigrations` asks interactively. Under
`--noinput` or in CI it answers "no" and emits `DROP COLUMN` + `ADD COLUMN`, which destroys every value
in that column on deploy. The migration applies cleanly and reports success.

**Fix:**

```python
operations = [
    migrations.RenameField(model_name="widget", old_name="titel", new_name="title"),
]
```

**Fine when:** the columns are genuinely unrelated, or the field type changed so drastically that a data
migration copies values across. **Confirm by checking whether a `RunPython` in the same migration copies
old → new** before reporting data loss.

Also grep CI config for `makemigrations --noinput`.

---

## Django: index builds lock the table

**Signal:** `migrations.AddIndex(...)`, `AddConstraint(...)`, or an `AlterField` increasing a
`CharField` `max_length`, in a normal atomic migration on a large table.

**Why it breaks:** plain `CREATE INDEX` takes an `ACCESS EXCLUSIVE` lock on PostgreSQL for the whole
build — every read and write blocks until it finishes. `AlterField` on a char column also drops and
rebuilds the `_like` index under the same lock.

**Fix:**

```python
from django.contrib.postgres.operations import AddIndexConcurrently

class Migration(migrations.Migration):
    atomic = False          # CONCURRENTLY cannot run in a transaction
    operations = [
        AddIndexConcurrently(
            model_name="event",
            index=models.Index(fields=["user", "-created_at"], name="event_user_created_idx"),
        ),
    ]
```

**Fine when:** under roughly 100k rows, or on a brand-new table created in the same migration — nothing
is reading it yet. **Do not push `CONCURRENTLY` there:** it is slower, cannot run in a transaction, and
a failure leaves an INVALID index that must be dropped manually.

Django's docs require `AddConstraintNotValid` and `ValidateConstraint` to be in two **separate**
migrations; doing both in one non-atomic migration can leave the database inconsistent.

---

## SQLAlchemy: lazy loading

**Signal:** `relationship("Child")` with no `lazy=` argument, plus code iterating
`for p in parents: p.children`.

**Why it breaks:** lazy select is the default for every `relationship()`. Each attribute access on each
loaded parent emits its own SELECT — the textbook N+1. **It is completely invisible in the source:** the
offending line is `p.children`, not a query call.

**Fix:**

```python
children = relationship("Child", lazy="raise_on_sql")          # make it loud

stmt = select(Parent).options(selectinload(Parent.children))   # load explicitly
stmt = select(Parent).options(raiseload("*"))                  # blanket guard on a read path
```

**Fine when:** the relationship is accessed on a single object — a detail view, a service loading one
row — or it is a many-to-one usually already in the identity map.

`lazy="raise"` is too aggressive for a codebase with mixed access patterns; **`lazy="raise_on_sql"` is
the safer switch**, since it still permits loads served from the identity map.

**Do not set `lazy="joined"` or `lazy="selectin"` at the mapper level as a blanket fix** — it makes every
query load the relation, including the ones that do not need it.

**Find it:** relationships never named in an `options()` call anywhere are the N+1 candidates.

---

## SQLAlchemy: `joinedload` on collections

**Signal:** `joinedload(Parent.children)` on a one-to-many or many-to-many, especially with `.limit()`,
`.offset()`, or `func.count()`. Missing `.unique()` on a 2.0-style result.

**Why it breaks:** a JOIN to a collection multiplies parent rows by child rows. In SQLAlchemy 2.0,
`session.execute(...).scalars()` raises `InvalidRequestError` unless you call `.unique()`. With `LIMIT`,
SQLAlchemy wraps the statement in a subquery to keep the limit correct — which silently changes the plan
and can be far slower than two queries.

**Fix:**

```python
stmt = select(Parent).options(selectinload(Parent.children)).limit(20)   # collections
parents = session.execute(stmt).scalars().all()

stmt = select(Child).options(joinedload(Child.parent))                   # many-to-one
```

**Fine when:** the target is many-to-one or one-to-one. `joinedload` is the **correct** tool there — one
JOIN, no extra round trip — and `selectinload` would be strictly worse. Also fine on a collection when
loading a handful of parents with no `LIMIT`.

`selectinload` has its own limits: it batches 500 parent primary keys per SELECT, and for composite
primary keys it needs tuple-IN syntax, which SQL Server does not support.

**Check the target:** `uselist=True` or `Mapped[list[...]]` means it is a collection.

---

## SQLAlchemy: session lifecycle

**Signal:** a `Session` created at module import or stored on a global. A `scoped_session` used from a
thread pool or async framework. Code reading ORM attributes after `session.commit()` outside the
session's `with` block.

**Why it breaks:** `expire_on_commit` defaults to **True**, so every object is expired at commit and the
next attribute read emits a fresh SELECT — an invisible query storm in web views — or raises
`DetachedInstanceError` if the session is closed. A Session shared across threads corrupts its identity
map and connection state. The docs are explicit: "Session per thread, AsyncSession per task."

**Fix:**

```python
SessionLocal = sessionmaker(engine, expire_on_commit=False)

def get_session():                  # one per request / task
    with SessionLocal() as session:
        yield session
```

**Fine when:** the module-level object is a `sessionmaker`/`async_sessionmaker` **factory** — that is
correct and expected. Only a module-level Session *instance* is the bug.

`expire_on_commit=True` is the right default for scripts and long-lived services that must see other
writers' changes after each commit; turning it off means objects can hold stale data across
transactions. `scoped_session` with a proper per-request `scopefunc` is valid in Flask.

---

## Alembic: autogenerate blind spots

**Signal:** an `alembic/versions/` directory with no `compare_server_default=True` in `env.py`, and
models using `server_default=`.

**Why it breaks:** Alembic's docs state `compare_server_default` is "off by default." Change or add a
server default in a model and autogenerate emits **nothing** — the migration looks clean and the database
keeps the old default forever. `compare_type` became True by default only in Alembic 1.12.0, so older
pins miss type changes silently too.

**Fix:** in both `run_migrations_online` and `run_migrations_offline`:

```python
context.configure(
    connection=connection,
    target_metadata=target_metadata,
    compare_type=True,
    compare_server_default=True,
    render_as_batch=True,          # required for SQLite ALTERs
)
```

**Fine when:** the project deliberately never uses server defaults — all defaults are Python-side. Then
leaving it off is defensible.

`compare_server_default` produces **false positives** on some backends: Postgres normalises `'0'` to
`'0'::integer`, so it can emit no-op `alter_column` calls you must delete by hand. That is still better
than silent misses, but tell the user to read every generated migration.

---

## Alembic: unnamed constraints and lost renames

**Signal:** an autogenerated migration with `op.drop_column(...)` + `op.add_column(...)` for what was a
rename. Or unnamed constraints — `UniqueConstraint('a','b')` with no `name=` — and no
`naming_convention` on `MetaData`.

**Why it breaks:** Alembic's docs list table renames, column renames, and anonymously named constraints
under what autogenerate "can't detect." A rename becomes drop + create, **which deletes the data**.
Unnamed constraints cannot be targeted by `drop_constraint`, so they are stuck. And in SQLite batch mode,
"unnamed CHECK constraints continue to be silently omitted from the table recreate operation" — they
vanish.

**Fix:**

```python
convention = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}
metadata = MetaData(naming_convention=convention)

op.alter_column("widget", "titel", new_column_name="title")   # hand-edit renames
```

**Fine when:** the `drop_column` is genuine — check git history for whether the model attribute was
renamed or actually deleted.

**Adding a naming convention to a project that already has migrations** will make autogenerate want to
rename every existing constraint. That is a one-time deliberate migration, not something to bolt on
casually — say so if you recommend it.

Batch mode is safe to leave on for non-SQLite backends; Alembic only applies the move-and-copy on SQLite.

---

## Sources

Django: [ForeignKey.on_delete](https://docs.djangoproject.com/en/5.2/ref/models/fields/#django.db.models.ForeignKey.on_delete),
[migrations](https://docs.djangoproject.com/en/5.2/topics/migrations/),
[postgres operations](https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/operations/),
[prefetch_related](https://docs.djangoproject.com/en/5.2/ref/models/querysets/#prefetch-related).
SQLAlchemy: [relationship loading](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html),
[session basics](https://docs.sqlalchemy.org/en/20/orm/session_basics.html).
Alembic: [autogenerate](https://alembic.sqlalchemy.org/en/latest/autogenerate.html),
[batch](https://alembic.sqlalchemy.org/en/latest/batch.html).
