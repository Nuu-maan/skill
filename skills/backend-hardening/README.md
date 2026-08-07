# backend-hardening

Audits a backend or API for security vulnerabilities, stability bugs, and design flaws, explains each
one in plain language, then fixes them on request.

Built for developers who are strong at frontend and learning backend — including people whose backend
was largely written by an AI. It reports before it edits, and every finding comes with a concrete
consequence and a way to check it yourself.

## What it covers

| Area | Examples |
| :--- | :--- |
| Authentication | Password hashing, sessions, cookie flags, JWT verification, password reset tokens, OAuth `state`, webhook signatures |
| Authorization | IDOR, missing role checks, mass assignment, multi-tenant leaks, client-trusted prices |
| Input and injection | SQL/NoSQL/command injection, path traversal, SSRF, template injection, deserialization, ReDoS |
| Data layer | Missing transactions, race conditions, idempotency, N+1, missing indexes, unbounded queries, migrations |
| API design | Status codes, error envelopes, CORS, pagination, security headers, body limits, over-fetching |
| Errors and resilience | Leaked stack traces, swallowed errors, missing timeouts, bad retries, resource leaks, graceful shutdown |
| Secrets and config | Committed `.env`, keys shipped to the browser, weak defaults, boot-time validation |
| Abuse and cost | Rate limiting, enumeration, uploads, unbounded spend on paid APIs |
| Operations | Logging PII, error tracking, dependency audits, container hardening, deploy safety |

Stack-specific checks for Node/TypeScript (Express, Nest, Fastify, Hono, Next.js), Python (Django,
DRF, FastAPI, Flask), PHP (Laravel, Symfony), Ruby (Rails, Sinatra), Go, Spring Boot, ASP.NET Core,
and Supabase/Firebase/Appwrite/PocketBase.

## Install

```bash
npx skills add https://qala.lol/numan/backend-hardening
```

Works in any agent that supports the [Agent Skills standard](https://agentskills.io) — Claude Code,
Cursor, Codex, Copilot, Gemini CLI, OpenCode, Goose, and others.

Or, in Claude Code, install the whole collection as a plugin:

```
/plugin marketplace add Nuu-maan/skill
/plugin install qala@qala
```

## Use

Ask for it in your own words — the skill triggers on requests like:

- "Review my backend before I deploy"
- "Is my API secure?"
- "Check this for vulnerabilities"
- "I built this with AI, what's wrong with it?"

Or invoke it directly in Claude Code:

```
/backend-hardening
```

You can scope it: `/backend-hardening focus on the auth routes` or
`/backend-hardening just check for exposed secrets`.

## What to expect

1. It identifies your stack from your files.
2. It runs a fast sweep of the twelve highest-signal checks.
3. It reads the relevant deep-dive checklists for what your code actually does.
4. It verifies each candidate finding against the real code path before reporting it.
5. You get a severity-ranked report with `file:line` evidence and a fixed-code example per finding.
6. It offers to fix, in priority order. It will not edit anything until you say yes.

If it finds nothing serious it says so, rather than padding the report.

## Claude Code variant

The canonical `SKILL.md` uses only the six fields in the Agent Skills spec, so it works everywhere.
If you use it exclusively in Claude Code, these frontmatter additions are worth considering:

```yaml
# Run the audit in an isolated subagent so the (long) reading phase does not
# consume your main conversation's context.
context: fork
agent: Explore
background: false
```

Trade-off: with `context: fork` the audit runs without your conversation history, and the fix phase
has to happen back in the main session. Good for large codebases, unnecessary for small ones.

```yaml
# Pre-approve read-only tools plus the ecosystem audit commands, so the audit
# runs without permission prompts.
allowed-tools: Read, Grep, Glob, Bash(npm audit*), Bash(pip-audit*), Bash(govulncheck*), Bash(bundle audit*)
```

```yaml
# Load automatically only when working in server-side code.
paths: ["**/api/**", "**/routes/**", "**/controllers/**", "**/handlers/**", "**/server/**"]
```

Do **not** add these fields to a copy you upload to claude.ai or the Skills API — those paths reject
non-spec keys with a hard error.

## Limitations

- Static review. It reads code; it does not run your app or attack it. Findings that depend on
  runtime configuration (environment variables set in a hosting dashboard, dashboard-configured
  database rules, network exposure) are flagged as unverifiable rather than guessed at.
- It is not a substitute for a penetration test on anything handling money, health data, or
  credentials at scale.
- Coverage is broad rather than exhaustive. It targets the defects that actually appear in
  small-team and AI-assisted backends.

## Files

```
backend-hardening/
├── SKILL.md                        # entry point: workflow, severity rubric, routing
├── references/                     # loaded only when relevant
│   ├── authentication.md
│   ├── authorization.md
│   ├── input-validation.md
│   ├── data-layer.md
│   ├── api-design.md
│   ├── errors-and-resilience.md
│   ├── secrets-and-config.md
│   ├── abuse-and-limits.md
│   ├── operations.md
│   └── fixing-safely.md
├── stacks/                         # one is loaded, matching the project
│   ├── node.md
│   ├── python.md
│   ├── php.md
│   ├── ruby.md
│   ├── go.md
│   ├── jvm-dotnet.md
│   └── baas.md
└── templates/
    └── report.md
```

Only `SKILL.md` enters context when the skill triggers. Everything else is read on demand, so the
depth costs nothing on a small audit.

## License

MIT
