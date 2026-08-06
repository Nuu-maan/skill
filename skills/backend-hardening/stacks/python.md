# Python

Django, Django REST Framework, FastAPI, Flask.

Framework-specific signals and defaults. Use alongside the `references/` files.

## Contents

- What to check first
- Django
- Django REST Framework
- FastAPI
- Flask
- SQLAlchemy
- Async pitfalls
- Grep list

---

## What to check first

1. `requirements.txt` / `pyproject.toml` — framework, ORM, validation, and whether a WSGI/ASGI server
   is present (running the dev server in production is a real and common finding).
2. Settings module: `DEBUG`, `ALLOWED_HOSTS`, `SECRET_KEY`, and how the production settings are
   selected.
3. Whether the app is started with `manage.py runserver`, `flask run`, or `uvicorn --reload` in a
   Dockerfile or Procfile — all are development servers, single-threaded and unhardened.

---

## Django

**`DEBUG = True` in production is Critical.** The error page shows source, local variables, and
settings — including `SECRET_KEY` and database credentials. Confirm how `DEBUG` is set; a common
pattern is `DEBUG = os.environ.get('DEBUG', True)`, which is truthy-by-default and also a string.

**`ALLOWED_HOSTS`.** `['*']` disables host validation, enabling host-header poisoning in password
reset emails (`authentication.md`).

**`SECRET_KEY`.** A key starting `django-insecure-` is the generated development key. It signs
sessions, password reset tokens, and CSRF tokens — if it is in the repo, all of those are forgeable.

**Run the built-in check.** `python manage.py check --deploy` finds most of the settings-level issues
directly. Report its output.

Settings worth confirming: `SECURE_SSL_REDIRECT`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`,
`SECURE_HSTS_SECONDS`, `SECURE_PROXY_SSL_HEADER` when behind a proxy, `X_FRAME_OPTIONS`.

**CSRF.** `@csrf_exempt` on a state-changing view is a finding unless the view uses token auth
instead of cookies. Grep for it.

**ORM injection.** `.raw()`, `.extra()`, and `connection.cursor().execute()` with an f-string or `%`
formatting. Note that `cursor.execute(sql % params)` is injection while `cursor.execute(sql, params)`
is safe — the difference is one character.

**Object-level authorization.** `get_object_or_404(Model, pk=pk)` without a user filter is the
standard IDOR shape. Correct form: `get_object_or_404(Model, pk=pk, owner=request.user)`.

**Admin.** Confirm `/admin/` is not publicly reachable, or is behind an IP allowlist / SSO. It is the
first URL anyone tries.

**Templates.** Django autoescapes; `|safe`, `mark_safe()`, and `{% autoescape off %}` disable it.

---

## Django REST Framework

**Default permission class.** Without `DEFAULT_PERMISSION_CLASSES` set to `IsAuthenticated`, the
default is `AllowAny` — every view is public unless it sets `permission_classes` itself. Deny by
default and opt out explicitly.

**Object permissions.** `permission_classes` alone checks view-level access. Object-level checks
require `has_object_permission` **and** a call to `check_object_permissions` — which generic views do
via `get_object()`, but a custom `retrieve`/`update` that fetches the object directly does not.

**Serializers.** `fields = '__all__'` is both mass assignment on input and over-fetching on output.
List fields explicitly, and mark computed or server-owned fields `read_only=True` — especially
`user`, `owner`, `role`, `is_staff`, `status`.

**Querysets.** `queryset = Model.objects.all()` on a ViewSet returns every row to every user. Override
`get_queryset()` to filter by `self.request.user`.

**Throttling.** `DEFAULT_THROTTLE_CLASSES` is usually unset. Add scoped throttles on auth endpoints.

---

## FastAPI

**Dependencies are the auth mechanism.** A route without `Depends(get_current_user)` is public. Apply
the dependency at the router or app level (`dependencies=[Depends(...)]`) rather than per route so a
new endpoint is protected by default.

**A dependency that returns `None` for an anonymous user is not a guard.** `get_current_user` must
raise `HTTPException(401)`, not return `None` — a handler that ignores the return value then runs
unauthenticated.

**Pydantic models validate the body, not authorization.** They also do not stop a field like `role`
from being accepted if you declared it. Use separate input and output models: `UserCreate` (no
`role`), `UserOut` (no `hashed_password`). Set `response_model` on every route — it acts as an output
allowlist.

**Docs.** `/docs` and `/openapi.json` are enabled by default and describe your whole API. Disable or
protect them in production if the API is not public.

**Path/query params.** Typed parameters are validated by type, but not by range. Add `Query(le=100)`
on `limit`, and validate that a `sort` string is in an allowlist.

**Background tasks.** `BackgroundTasks` runs in the same process after the response — it is lost on
restart and does not scale. Not a replacement for a queue.

---

## Flask

**Nothing is on by default.** No auth, no CSRF, no rate limiting, no security headers, no
serialization allowlist. Check that each is added explicitly: `Flask-Login`/custom decorator,
`Flask-WTF` CSRF, `Flask-Limiter`, `Talisman`.

**`app.run(debug=True)`** exposes the Werkzeug debugger, which offers an interactive Python console
to anyone who can reach it. Critical if reachable.

**`SECRET_KEY`** signs the session cookie. The Flask session is **signed, not encrypted** — anything
stored in it is readable by the client. Never put anything secret in `session`.

**Decorator order.** `@app.route` must be the outermost decorator; an auth decorator placed above it
does not run. This silently disables the check.

**`render_template_string`** with user input is server-side template injection → RCE. Grep for it.

**`send_file` / `send_from_directory`** with a user-supplied path is path traversal. Use
`safe_join` / `send_from_directory` with a fixed base and validate the filename.

---

## SQLAlchemy

`text()` with an f-string is injection; `text("... :id").bindparams(id=id)` or passing a params dict
is safe. `session.execute(f"...")` is injection.

Check `expire_on_commit`, and that sessions are scoped per request and closed in a `finally` —
a leaked session holds a connection.

Eager loading (`selectinload`, `joinedload`) prevents N+1; without it, accessing a relationship in a
loop issues a query per row.

---

## Async pitfalls

**Blocking calls in an async handler.** `requests.get`, `time.sleep`, a sync database driver, or heavy
CPU work inside `async def` blocks the entire event loop — every concurrent request stalls, not just
this one. Use `httpx.AsyncClient`, `asyncio.sleep`, an async driver, or `run_in_threadpool`.

**Mixing sync and async ORM sessions** produces subtle connection errors under load.

**`asyncio.create_task` without keeping a reference** can be garbage collected mid-flight; the task
silently disappears.

---

## Grep list

```bash
# config
rg -n "DEBUG\s*=|ALLOWED_HOSTS|SECRET_KEY|django-insecure"
rg -n "app\.run\(.*debug\s*=\s*True|--reload"

# injection
rg -n "\.raw\(|\.extra\(|execute\(f['\"]|execute\(.*%\s*\(|text\(f['\"]"
rg -n "os\.system|subprocess.*shell\s*=\s*True|eval\(|exec\("
rg -n "render_template_string|Template\("

# authorization
rg -n "get_object_or_404|objects\.get\(|objects\.filter\(pk="
rg -n "fields\s*=\s*['\"]__all__['\"]|permission_classes|AllowAny"
rg -n "csrf_exempt"

# mass assignment
rg -n "\*\*request\.(data|POST|json)|\*\*payload"

# deserialization / secrets
rg -n "pickle\.loads|yaml\.load\((?!.*SafeLoader)"
rg -n "verify\s*=\s*False"
```
