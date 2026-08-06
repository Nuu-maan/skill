# PHP

Laravel, Symfony, and plain PHP.

Framework-specific signals and defaults. Use alongside the `references/` files.

## Contents

- What to check first
- Laravel
- Symfony
- Plain PHP
- PHP language traps
- Grep list

---

## What to check first

1. `composer.json` — framework and version, and whether the version is still receiving security fixes.
2. `.env` — is it committed? Is `APP_DEBUG` false and `APP_ENV` production? Is `APP_KEY` set?
3. Whether the document root points at `public/` and not the project root. If the project root is
   served, `.env`, `composer.json`, and `storage/` are downloadable over HTTP. That is Critical.

---

## Laravel

**`APP_DEBUG=true` in production is Critical.** Ignition's error page exposes environment variables —
including `APP_KEY`, database credentials, and every API key — to anyone who triggers an error.

**`APP_KEY`.** Encrypts cookies and signs sessions. If it is missing, default, or committed, session
and cookie forgery follow.

**Mass assignment.** Check every model for `$fillable` (allowlist, good) versus `$guarded = []`
(everything writable, bad). `Model::create($request->all())` and `$model->update($request->all())`
let the client set any column. Use `$request->validated()` from a FormRequest, which returns only the
validated keys.

**Authorization.** Laravel has Policies and Gates; the common failure is not using them.
`Post::find($id)` with no `$this->authorize('view', $post)` is IDOR. Prefer route-model binding with
a scoped query, or `$request->user()->posts()->findOrFail($id)`, which cannot be forgotten.

**Query injection.** Eloquent parameterizes, but these do not: `DB::raw()`, `whereRaw`, `havingRaw`,
`orderByRaw`, `selectRaw` with interpolation, and `DB::statement`. `orderByRaw($request->sort)` is a
common one — allowlist the column.

**Blade.** `{{ }}` escapes; `{!! !!}` does not. Grep for `{!!` on user content.

**Routes.** `php artisan route:list` gives the full route table with middleware per route — the
fastest way to find routes missing `auth`. Report any state-changing route without an auth middleware.

**Config caching.** `env()` returns `null` outside config files once `config:cache` has run. A call
to `env()` in application code works locally and silently becomes `null` in production — check for
`env(` outside `config/`.

**Storage.** Files in `storage/app/public` are exposed via the symlink; `storage/app` is not. Confirm
uploads land where intended.

**Queues.** `QUEUE_CONNECTION=sync` runs jobs inline in the request — check it is not `sync` in
production if the code assumes background execution.

---

## Symfony

**`APP_ENV=prod` and `APP_DEBUG=0`.** The profiler and web debug toolbar must not be reachable in
production; the profiler exposes requests, queries, and configuration.

**Security configuration.** `security.yaml` is the whole access-control story. Check `access_control`
rules cover every path prefix, that the order is correct (first match wins), and that there is a
default-deny rule at the end. A path not matched by any rule is public.

**Voters** are the object-level mechanism. `$this->denyAccessUnlessGranted('EDIT', $post)` must
actually be called in the controller — a voter that is never invoked does nothing.

**Doctrine.** `createQuery` with string concatenation is injection; use `setParameter`. `findOneBy`
with a raw request array can be manipulated.

**Serialization groups** control what leaves the API. Without groups, the serializer may expose the
whole entity including relations.

---

## Plain PHP

**Every request parameter is a string from an untrusted source.** `$_GET`, `$_POST`, `$_COOKIE`,
`$_REQUEST`, `$_FILES`, `$_SERVER['HTTP_*']`.

- **SQL:** use PDO prepared statements with real prepares
  (`PDO::ATTR_EMULATE_PREPARES => false`) and bound parameters. Never interpolate into the query.
- **Includes:** `include $_GET['page']` is local/remote file inclusion → RCE. Allowlist the page
  names; never build a path from input.
- **Uploads:** never trust `$_FILES['x']['type']` or the extension. Move the file out of the web
  root, rename it, and verify the content type from the bytes. A `.php` in a served directory is RCE.
- **Sessions:** `session_regenerate_id(true)` on login. Set `session.cookie_httponly`,
  `session.cookie_secure`, `session.cookie_samesite`, and `session.use_strict_mode=1` in `php.ini`
  or via `session_set_cookie_params`.
- **Output:** `htmlspecialchars($v, ENT_QUOTES, 'UTF-8')` on everything echoed. There is no
  autoescaping.
- **Errors:** `display_errors=Off` and `log_errors=On` in production.
- **CSRF:** there is none by default. Add a per-session token to every state-changing form and verify
  it with `hash_equals`.

---

## PHP language traps

**Loose comparison.** `==` performs type juggling. `"0e123" == "0e456"` is true (both parse as
scientific-notation zero), which historically bypassed hash comparisons. `"abc" == 0` was true before
PHP 8. Always `===`, and `hash_equals()` for secrets.

**`in_array` and `array_search`** default to loose comparison — pass `true` as the third argument.

**Null-byte and encoding issues** in path handling; validate rather than sanitize.

**`extract($_POST)`** creates local variables from request data — it can overwrite anything in scope.
Never use it on untrusted input.

**`unserialize()`** on user data enables object injection through magic methods. Use `json_decode`.

---

## Grep list

```bash
# config
rg -n "APP_DEBUG|APP_ENV|display_errors|APP_KEY"

# injection
rg -n "DB::raw|whereRaw|orderByRaw|havingRaw|selectRaw|DB::statement"
rg -n "\\\$db->query\(|mysqli_query\(|->exec\(" 
rg -n "shell_exec|exec\(|system\(|passthru|popen|eval\("

# mass assignment
rg -n "\\\$guarded\s*=\s*\[\]|->update\(\\\$request->all\(\)|::create\(\\\$request->all\(\)"

# authorization
rg -n "::find\(|::findOrFail\(|firstOrFail\("
rg -n "authorize\(|denyAccessUnlessGranted|Gate::"

# output and includes
rg -n "\{!!|echo \\\$_(GET|POST|REQUEST)"
rg -n "include\s*\\\$|require\s*\\\$"

# misc
rg -n "unserialize\(|extract\(|==\s*\\\$_(GET|POST)"
```

Also run, if the tooling is available: `php artisan route:list`, `composer audit`.
