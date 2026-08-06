# JVM and .NET

Spring Boot (Java/Kotlin) and ASP.NET Core (C#).

Framework-specific signals and defaults. Use alongside the `references/` files.

## Contents

- Spring Boot — what to check first
- Spring Security
- Spring data access
- Spring configuration
- ASP.NET Core — what to check first
- ASP.NET authorization
- Entity Framework
- ASP.NET configuration
- Grep list

---

## Spring Boot — what to check first

1. `pom.xml` / `build.gradle` — Spring Boot version (is it still supported?), whether
   `spring-boot-starter-security` is present at all, and the database driver.
2. `application.properties` / `application.yml` per profile — which profile is active in production.
3. Whether Actuator is on the classpath and which endpoints are exposed.

---

## Spring Security

**Absent entirely.** If `spring-boot-starter-security` is not a dependency, there is no
authentication and every endpoint is public. That is the first thing to establish.

**Permissive configuration.** In the security filter chain, look for:

```java
.authorizeHttpRequests(auth -> auth.anyRequest().permitAll())   // everything public
.csrf(csrf -> csrf.disable())                                   // fine for stateless token APIs,
                                                                // Critical for cookie-session apps
```

Rules are evaluated in order and the first match wins, so a broad `permitAll()` placed before a
specific `hasRole()` makes the later rule dead. Read the chain in order. End with
`.anyRequest().authenticated()` so a new endpoint is protected by default.

**Method security.** `@PreAuthorize` only works if method security is enabled
(`@EnableMethodSecurity`). Without it the annotations are silently inert — verify the annotation is
enabled, not just present.

**Object-level checks.** `@PreAuthorize("hasRole('USER')")` says nothing about *which* record. The
IDOR shape is `repository.findById(id)` with no owner condition. Use
`findByIdAndOwner(id, principal)` or `@PostAuthorize("returnObject.owner == authentication.name")`
— noting `@PostAuthorize` loads the object first, so it is unsuitable if loading itself is expensive
or has side effects.

**Actuator.** `/actuator/env`, `/heapdump`, `/threaddump`, and `/mappings` expose configuration
including secrets. Expose only `health` and `info` publicly; secure the rest.
`management.endpoints.web.exposure.include=*` is a finding.

**CORS.** `@CrossOrigin` with no arguments allows all origins on that controller. Configure CORS
centrally with an explicit origin list.

---

## Spring data access

**JPQL/SQL injection:**

```java
em.createQuery("FROM User WHERE email = '" + email + "'");            // injection
em.createQuery("FROM User WHERE email = :email").setParameter(...);   // safe

@Query(value = "SELECT * FROM users WHERE name = '" + ... , nativeQuery = true)   // injection
```

Spring Data derived query methods (`findByEmail`) are safe. `Sort`/`Pageable` values that come from
the request must be allowlisted — an unvalidated sort property can reference unintended columns.

**Mass assignment.** Binding a request body directly to a JPA entity lets the client set any field,
including the id and the owner. Use a separate DTO per request, and `@Valid` with Bean Validation
annotations. For `@ModelAttribute` binding, restrict with `@InitBinder` `setAllowedFields`.

**Over-fetching.** Returning entities from controllers serializes lazy relations (or throws), and
exposes every field. Return DTOs. Annotate password/token fields `@JsonIgnore` as a backstop.

**Transactions.** `@Transactional` is proxy-based: it does nothing when the method is called from
within the same class (self-invocation), and by default only rolls back on unchecked exceptions.
Both are common sources of "the transaction did not roll back".

**Lazy loading** outside a transaction throws `LazyInitializationException`; `open-in-view` (on by
default) hides this while causing N+1 queries in the view layer. Consider disabling it and fetching
explicitly.

---

## Spring configuration

- `server.error.include-stacktrace=never` and `include-message=never` in production.
- `spring.jpa.hibernate.ddl-auto` must be `validate` or `none` in production, never `update` or
  `create-drop`.
- `spring.h2.console.enabled` must be false outside development.
- Secrets in `application.properties` committed to the repo — move to environment variables or a
  secret manager.
- `spring.jpa.show-sql=true` logs queries with parameters.
- Deserialization: Jackson polymorphic typing (`enableDefaultTyping`, `@JsonTypeInfo` with `Id.CLASS`)
  on untrusted input is a remote-code-execution vector.

---

## ASP.NET Core — what to check first

1. `*.csproj` — target framework (is it still supported?) and packages.
2. `Program.cs` — middleware order and endpoint mapping.
3. `appsettings.json` and `appsettings.Production.json` — connection strings and secrets committed?

---

## ASP.NET authorization

**Middleware order is enforced and easy to get wrong.** `UseAuthentication()` must come before
`UseAuthorization()`, and both must come before `MapControllers()`/endpoint execution. Wrong order
means authorization runs without an identity.

**Default is anonymous.** Without `[Authorize]`, an action is public. Prefer a fallback policy so
everything requires auth and `[AllowAnonymous]` opts out:

```csharp
builder.Services.AddAuthorization(o =>
    o.FallbackPolicy = new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build());
```

**`[Authorize]` is authentication plus role, not ownership.** The IDOR shape is
`_db.Orders.FindAsync(id)`; scope it: `_db.Orders.FirstOrDefaultAsync(o => o.Id == id && o.UserId == userId)`.
Get the user id from `User.FindFirstValue(ClaimTypes.NameIdentifier)`, never from the request body.

**Antiforgery.** Required for cookie-authenticated forms. `[IgnoreAntiforgeryToken]` on a
state-changing action is a finding for cookie auth.

---

## Entity Framework

```csharp
_db.Users.FromSqlRaw($"SELECT * FROM Users WHERE Name = '{name}'");   // injection
_db.Users.FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {name}");  // parameterized
_db.Database.ExecuteSqlRaw(sql + input);                              // injection
```

`FromSqlRaw` with concatenation is the sink; `FromSqlInterpolated` and `FromSqlRaw` with explicit
parameters are safe. The two names are confusingly similar — read carefully.

**Over-posting (mass assignment):** binding a request directly to an entity lets the client set
`IsAdmin` or `UserId`. Use view models / DTOs and `[Bind]` allowlists.

**Tracking and N+1:** use `AsNoTracking()` for reads, `Include()` for relations, and check for
queries inside loops. `ToList()` before a `Where` pulls the whole table into memory — check where
enumeration actually happens.

---

## ASP.NET configuration

- `app.UseDeveloperExceptionPage()` must be inside an `IsDevelopment()` check.
- `UseHsts()` and `UseHttpsRedirection()` in production.
- Connection strings and API keys belong in environment variables or a secret store, not
  `appsettings.json` in the repo. Use User Secrets locally.
- `AddCors` with `AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod()` — and note that combining
  `AllowAnyOrigin` with `AllowCredentials` throws at startup, so a codebase that "works" may be
  reflecting origins instead. Check how.
- Swagger UI enabled in production on a non-public API.
- `BinaryFormatter` on untrusted data — remote code execution, removed in recent .NET for this
  reason.

---

## Grep list

```bash
# Spring
rg -n "permitAll|csrf\(\)\.disable|\.disable\(\)" --type java
rg -n "createQuery\(\"[^\"]*\"\s*\+|nativeQuery\s*=\s*true" --type java
rg -n "ddl-auto|include-stacktrace|exposure.include|h2.console"
rg -n "@CrossOrigin|@PreAuthorize|EnableMethodSecurity" --type java
rg -n "findById\(" --type java

# ASP.NET
rg -n "FromSqlRaw|ExecuteSqlRaw" --type cs
rg -n "UseAuthentication|UseAuthorization|MapControllers" --type cs
rg -n "\[AllowAnonymous\]|IgnoreAntiforgeryToken" --type cs
rg -n "AllowAnyOrigin|UseDeveloperExceptionPage|BinaryFormatter" --type cs
rg -n "FindAsync\(|\.Find\(" --type cs
```

Also run if available: `mvn dependency-check:check` / `gradle dependencyCheckAnalyze`,
`dotnet list package --vulnerable --include-transitive`.
