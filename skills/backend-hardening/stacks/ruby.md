# Ruby

Rails and Sinatra.

Framework-specific signals and defaults. Use alongside the `references/` files.

## Contents

- What to check first
- Rails
- Sinatra
- Grep list

---

## What to check first

1. `Gemfile` / `Gemfile.lock` — Rails version and whether it is still supported. Run
   `bundle audit check --update` if available.
2. `config/environments/production.rb` — `consider_all_requests_local`, `force_ssl`, log level.
3. `config/credentials.yml.enc` and `master.key` — the key must **not** be committed. Grep
   `.gitignore` for it.
4. Run `bin/rails routes` for the full route table; it is the fastest way to enumerate what is
   exposed.

---

## Rails

**Strong parameters.** `params.require(:user).permit(:name, :email)` is the allowlist. Findings:

- `params[:user]` passed to `create`/`update` without `permit` (older code, or `permit!`).
- `.permit!` anywhere — it permits everything.
- A `permit` list that includes `:role`, `:admin`, `:user_id`, `:credits`, or `:verified`.

**Authorization.** Rails ships authentication helpers but no authorization. Check for Pundit or
CanCanCan, and — more importantly — that they are actually invoked. Pundit's
`after_action :verify_authorized` is what turns "we have Pundit" into "every action is checked".

The IDOR shape here is `Post.find(params[:id])`. Correct: `current_user.posts.find(params[:id])`,
which raises `RecordNotFound` (404) for someone else's record without a separate check.

**SQL injection.** ActiveRecord parameterizes when you use the hash or array form. These do not:

```ruby
User.where("name = '#{params[:name]}'")        # injection
User.where("name = ?", params[:name])          # safe
User.order(params[:sort])                      # injection via ORDER BY
User.find_by_sql("... #{x}")                   # injection
```

Also `pluck`, `group`, `having`, `joins`, and `select` accept raw SQL strings. Allowlist any
user-supplied column or direction.

**Mass assignment via nested attributes.** `accepts_nested_attributes_for` combined with a permissive
`permit` lets a caller create or modify associated records — including reassigning ownership.

**Views.** ERB escapes by default; `raw`, `html_safe`, and `<%== %>` do not. `.html_safe` on user
content is stored XSS.

**CSRF.** `protect_from_forgery` is on by default in Rails; `skip_before_action :verify_authenticity_token`
on a state-changing controller is a finding unless the endpoint uses token auth. API-only apps do not
include it at all — confirm those endpoints are not cookie-authenticated.

**Callbacks.** `before_action :authenticate_user!` with `skip_before_action` in a subclass is the
usual way a route ends up public. Check subclasses and `only:`/`except:` lists.

**Serialization.** `render json: @user` renders every column. Use a serializer, `jbuilder`, or
`only:`. Confirm `password_digest` and tokens are excluded.

**`to_json` on a collection with includes** pulls in associated records wholesale — check what the
associations expose.

**Secrets.** `Rails.application.credentials` is the right place; `master.key` must be in
`.gitignore` and provided via `RAILS_MASTER_KEY` in production. `secret_key_base` in a committed
YAML file is a leaked signing key.

**Production settings:** `config.force_ssl = true`, `config.consider_all_requests_local = false`,
`config.log_level = :info` (`:debug` logs parameters, which can include credentials — Rails filters
`password` by default via `filter_parameters`; confirm the filter list covers tokens and card data).

**Background jobs.** Sidekiq's web UI is often mounted without authentication. Check
`config/routes.rb` for `mount Sidekiq::Web` and require auth around it.

---

## Sinatra

Almost nothing is on by default. Check for each explicitly:

- Authentication — usually a `before` filter. Confirm the filter's path pattern covers every route,
  including ones added later. `before '/admin/*'` does not cover `/admin` itself.
- `sinatra-contrib`'s `Rack::Protection` — included with `Sinatra::Base`? Classic style enables some
  protections; modular style may not. Verify rather than assume.
- CSRF — `Rack::Protection::AuthenticityToken` must be enabled and tokens rendered in forms.
- Sessions — `enable :sessions` uses a cookie store; set a strong `session_secret` from the
  environment, and remember the cookie is signed but readable.
- ERB escaping — Sinatra's ERB does **not** escape by default. Use `Erubi` with `escape: true`, or
  call `h()` on every interpolated value.
- `send_file` with a user-supplied path is traversal.

---

## Grep list

```bash
# mass assignment
rg -n "permit!|params\[:\w+\]\)" app/
rg -n "permit\(.*(:role|:admin|:user_id|:credits)"

# authorization
rg -n "\.find\(params\[:id\]\)|find_by_id\(params"
rg -n "skip_before_action|skip_authorization|verify_authorized"

# injection
rg -n 'where\("[^"]*#\{|order\(params|find_by_sql|execute\("'
rg -n "system\(|`|%x\(|Open3|eval\("

# output
rg -n "html_safe|raw\(|<%==" app/views/

# config and secrets
rg -n "consider_all_requests_local|force_ssl|log_level"
rg -n "secret_key_base" config/
rg -n "master.key" .gitignore

# csrf and mounts
rg -n "verify_authenticity_token|mount .*::Web" config/ app/
```

Also run if available: `bundle audit check --update`, `brakeman` (a Rails-specific static analyzer —
its output is a strong starting point, but verify each finding against the code before reporting it).
