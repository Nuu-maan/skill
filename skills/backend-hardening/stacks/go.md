# Go

net/http, Gin, Echo, Fiber, Chi.

Framework-specific signals and defaults. Use alongside the `references/` files.

## Contents

- What to check first
- Error handling
- Routing and middleware
- Database
- Concurrency
- HTTP server and client defaults
- Framework notes
- Grep list

---

## What to check first

1. `go.mod` — Go version and dependencies. Run `govulncheck ./...` if available.
2. The router setup — which middleware wraps which route group.
3. `go vet ./...` and, if present, `staticcheck` / `golangci-lint`. Real bugs, cheap to run.

---

## Error handling

**Ignored errors are the dominant bug class in Go.**

```go
result, _ := doSomething()        // error discarded
json.Unmarshal(body, &input)      // error unchecked — input may be zero-valued
tx.Commit()                       // unchecked — the transaction may not have committed
defer resp.Body.Close()           // fine, but the earlier err from the request may be unchecked
```

**Why it breaks:** an unchecked `Unmarshal` leaves the struct zero-valued and the handler proceeds
with `UserID: 0` or an empty string. An unchecked `Commit` means data silently was not saved.

**Fix:** handle every error. `errcheck` (in golangci-lint) finds them mechanically. A deliberate
discard should be `_ = f()` with a comment saying why.

**Error responses:** `http.Error(w, err.Error(), 500)` returns the internal error text to the client —
often including SQL, file paths, or connection strings. Log the error, return a generic message.

**Missing return after an error write:**

```go
if err != nil {
    http.Error(w, "bad", 400)
    // no return — execution continues into the success path
}
```

This is the same auth-bypass shape as elsewhere; check every error branch has a `return`.

---

## Routing and middleware

Middleware in Go is explicit composition — there is no framework magic applying it for you. Read the
route registrations and confirm each protected route is inside a group that has the auth middleware
applied. A route registered on the base mux instead of the authenticated subrouter is public.

**Values passed via context** (`context.WithValue`) are typed `any`. A type assertion
`ctx.Value(userKey).(string)` panics if the middleware did not run — or, with the comma-ok form,
silently yields the zero value and the handler treats an unauthenticated caller as user `""`. Always
use the comma-ok form **and** reject when it fails. Use an unexported key type to prevent collisions.

**`http.ServeMux` path matching** — be precise about trailing slashes and prefix registration; a
pattern ending in `/` matches the whole subtree, which can expose more than intended.

---

## Database

`database/sql` parameterizes when you pass arguments:

```go
db.Query("SELECT * FROM users WHERE id = $1", id)     // safe
db.Query("SELECT * FROM users WHERE id = " + id)      // injection
fmt.Sprintf("SELECT ... WHERE name = '%s'", name)     // injection
```

**`sql.DB` is a pool, not a connection.** Create it once and share it. Configure
`SetMaxOpenConns`, `SetMaxIdleConns`, and `SetConnMaxLifetime` — the defaults are unlimited open
connections, which exhausts the database under load.

**`Rows` must be closed.** `defer rows.Close()` and check `rows.Err()` after the loop — a failure
mid-iteration is otherwise invisible.

**`QueryRow().Scan()`** returns `sql.ErrNoRows` for a missing row; treating that as a generic error
gives a 500 where a 404 belongs.

**GORM:** `Raw()` and `Exec()` with formatted strings are injection sinks; use `?` placeholders.
`db.First(&user, id)` without a user-scoped `Where` is the IDOR shape. Be careful with `Updates()`
taking a `map[string]any` built from request data — that is mass assignment.

---

## Concurrency

**Data races.** A shared map or struct written from multiple goroutines without a mutex. Go maps are
not safe for concurrent write and will panic (`concurrent map writes`), taking the process down.
Run the race detector: `go test -race ./...`.

**Goroutine leaks.** A goroutine blocked forever on a channel send/receive with no reader, or one
started per request that outlives it. Memory grows until OOM. Every goroutine needs a way to exit —
usually a context.

**Context not propagated.** `context.Background()` inside a request handler ignores client
cancellation and any deadline, so work continues after the client is gone. Pass `r.Context()` through
to every database call and outbound request.

**Loop variable capture** — in older Go versions, `for _, v := range xs { go func(){ use(v) }() }`
captures the same variable. Confirm the Go version in `go.mod`; if it is before the semantics change,
this is a real bug.

**Unbounded goroutines per request** — spawning one per item in a user-controlled list is a denial of
service. Bound with a worker pool or semaphore.

---

## HTTP server and client defaults

**The zero-value `http.Server` has no timeouts.** A slow client holds a connection indefinitely
(Slowloris). Always set them:

```go
srv := &http.Server{
    Addr:              addr,
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       15 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       60 * time.Second,
    MaxHeaderBytes:    1 << 20,
}
```

`http.ListenAndServe(addr, mux)` uses that zero-value server — no timeouts at all.

**`http.DefaultClient` has no timeout either.** Create a client with an explicit `Timeout`, or use
`http.NewRequestWithContext` with a deadline. Reuse one client; creating one per request leaks
connections.

**Request bodies are unbounded.** Wrap with `http.MaxBytesReader(w, r.Body, n)` before decoding, and
use `json.NewDecoder(r.Body)` with `DisallowUnknownFields()` to reject extra keys (mass assignment).

**TLS:** `InsecureSkipVerify: true` disables certificate verification entirely. Grep for it.

---

## Framework notes

**Gin:** `c.ShouldBindJSON` returns an error you must check; `c.BindJSON` aborts with 400
automatically, which is fine but changes control flow. Validate with binding tags (`binding:"required,max=100"`).
`gin.Default()` includes Logger and Recovery; `gin.New()` includes neither — without Recovery, a panic
kills the process. Confirm `c.Abort()` is called in auth middleware, not just `c.JSON(401, ...)` —
without `Abort`, the handler still runs.

**Echo:** middleware order matters; `e.Use` applies to routes registered after it. `c.Bind` respects
struct tags — use separate request structs, not database models.

**Fiber:** built on fasthttp, so `context.Context` semantics and some `net/http` middleware do not
apply. `c.Body()` is only valid during the handler — do not retain it in a goroutine.

**Chi:** compose middleware per `Route`/`Group`. Verify each group's middleware stack.

---

## Grep list

```bash
# ignored errors and injection
rg -n ", _ :?= " --type go
rg -n 'Sprintf\("(SELECT|INSERT|UPDATE|DELETE)' --type go
rg -n 'db\.(Query|Exec)\([^,)]*\+' --type go
rg -n "exec\.Command\(\"(sh|bash|cmd)\"" --type go

# server/client defaults
rg -n "ListenAndServe\(|http\.Get\(|http\.Post\(|DefaultClient" --type go
rg -n "InsecureSkipVerify" --type go

# context and auth
rg -n "context\.Background\(\)|context\.TODO\(\)" --type go
rg -n "ctx\.Value\(" --type go
rg -n "c\.JSON\(http\.StatusUnauthorized" --type go   # check Abort/return follows

# errors leaking
rg -n "http\.Error\(w, err\.Error\(\)" --type go
```

Also run if available: `go vet ./...`, `go test -race ./...`, `govulncheck ./...`,
`golangci-lint run`.
