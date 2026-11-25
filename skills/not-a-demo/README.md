# not-a-demo

**Your AI built an app. It looks finished in the screenshot. Then someone uses it.**

The list is blank when there's no data. The button does nothing for two seconds. "Something went
wrong." The delete has no confirm. The accent is the same indigo as every other AI app, the headline
says "Unlock your potential", and the Supabase table is readable by anyone with the public key.

not-a-demo is an agent skill that closes that gap. It makes AI-built web apps **look designed** and
**behave like shipped software**: every state handled, every action answered, every word written
for a person, and every route checking who's asking.

## Why it's different

Most "anti-slop" skills fix how a landing page looks. This one covers the whole app:

| | |
| :--- | :--- |
| **A design direction, not a ban list** | Before the first screen, the agent writes a short `DESIGN.md`: theme, one accent, type, density, a layout per screen type. Every screen is checked against it. Ban lists breed new clichés; a written direction doesn't. |
| **Product UI first** | Dashboards, settings, tables, inboxes, editors: the screens where AI output is least original and other skills say least. |
| **Look and behavior together** | Visual rules plus loading, empty and error states, optimistic updates, autosave, keyboard and phone support. |
| **Security built in** | Server-side ownership checks, no secrets in the bundle, row-level security, safe sessions, headers. The holes AI-built apps keep shipping. |
| **Fits the app** | Rule zero: it doesn't bolt Redis, audit logs or referral pages onto a todo app. Growth and infra features are suggested, not added. |
| **Every ban has a replacement** | "Never indigo-500" comes with what to use instead. Telling a model only what to avoid tends to make it do that thing more. |
| **Checks you can run** | A set of ripgrep searches finds AI tells, copy tells and security holes in the code. No install, no CLI. |
| **Sourced** | Every rule traces to a designer, an engineer, a standard or a real incident. See [SOURCES.md](SOURCES.md). |

## What it covers

| Area | Examples |
| :--- | :--- |
| Design direction | The `DESIGN.md` template, choosing an accent and a tinted neutral, picking layouts by job (triage, create, set up, decide, monitor), defaults when there's no brand, and the second-wave clichés to avoid (cream backgrounds, Instrument Serif, one italic word in the headline) |
| Visual design | 12-step color scales in OKLCH, 4px spacing, about four type sizes per screen, letter-spacing by size, layered shadows, radius sets, dark mode elevation and muted colors, grain on gradients, baseline alignment, Delete kept away from Save, icons, KPI tiles that aren't four identical cards |
| Motion | When not to animate, durations per element, easing curves, entering from 0.95 not 0, steps that slide the way you're going, animating to `height: auto`, rubber-band drag, springs that keep your flick's speed, error shakes, reduced motion, hover only on real pointers |
| Interaction | Focus rings, menu and dialog keyboard behavior, tooltip delays, toast timing, drag and drop, hit areas, loading indicators that don't flash |
| UX patterns | Optimistic UI, skeletons, command palette, sidebar, empty states, autosave, debounce, validation, bulk actions, data tables, filters in the URL, shortcuts, plan limits, destructive-action tiers, error recovery, and 15 UX laws (Pareto to social proof) turned into layout decisions |
| UX writing | Verb + object buttons, errors that say how to fix it, confirmations that name the thing, sentence case, and a table of AI-copy tells with what to write instead (including "quiet", "real" and two-word sentences) |
| Security | Ownership checks in every handler, secrets behind `NEXT_PUBLIC_`/`VITE_`, Supabase RLS and Firebase rules, mass assignment, injection, XSS, CSRF, CORS, a starter CSP and header set, uploads, SSRF, webhooks, slopsquatting, prompt injection and AI cost abuse |
| SaaS essentials | The full auth flow, password reset done right, email scanners that burn magic links, account deletion, billing webhooks and failed payments, money as integers, time zones, backups you've restored, safe migrations, favicons, Open Graph, legal pages |
| Frontend | Prefetch on intent, instant route shells, back/forward cache, scroll restoration, URL as state, surviving deploys, offline, forms that work before hydration, a CSS baseline, hydration bugs |
| Backend | Pagination, N+1, indexes, pooling, background jobs, timeouts, transactions, one error shape, caching with an invalidation plan, one log line per request |

## Use cases

- **Starting a new app.** "Build me a CRM for a small agency." The agent writes `DESIGN.md` first,
  then builds screens with every state, real copy and server-side checks from the start.
- **Rescuing a vibe-coded app.** "This looks AI-generated, fix it." The agent runs the slop
  checks, records a design direction, and replaces defaults with decisions.
- **Adding a feature.** "Add billing" or "add file uploads." It loads only the reference that
  applies: webhooks, idempotency and failed payments, or upload validation.
- **Before you launch.** "Is this ready to ship?" It walks every screen (loading, empty, error,
  slow network, phone, keyboard), tries to read user B's data as user A, and runs the pre-ship
  security checklist.
- **A copy pass.** "The wording feels off." It rewrites buttons, errors, empty states and
  confirmations against the writing rules.
- **A design review.** "Review this dashboard." It checks the screen against `DESIGN.md` and the
  visual rules, then reports what to change.

## Install

```bash
npx skills add https://qala.lol/numan/not-a-demo
```

Or straight from GitHub:

```bash
npx skills add Nuu-maan/skill --skill not-a-demo
```

Works in any agent that supports the [Agent Skills standard](https://agentskills.io): Claude Code,
Cursor, Codex, Copilot, Gemini CLI, OpenCode, Goose and others.

In Claude Code you can install the whole collection as a plugin:

```
/plugin marketplace add Nuu-maan/skill
/plugin install qala@qala
```

**Manual install:** copy the `not-a-demo` folder into your agent's skills folder
(`~/.claude/skills/` for Claude Code, or `.claude/skills/` in a project). For an agent without skill
support, copy the folder into your repo and add one line to `AGENTS.md`:

```
When building or changing UI, follow skills/not-a-demo/SKILL.md.
```

## Use

Build as usual. The skill triggers on requests like:

- "Build me a dashboard for my Stripe data"
- "Make this not look AI-generated"
- "Add a settings page with account deletion"
- "Why does this app feel unfinished?"
- "Is this safe to launch?"

Or invoke it directly in Claude Code:

```
/not-a-demo
```

You can scope it: `/not-a-demo just the copy` or `/not-a-demo review the security before launch`.

## What to expect

1. **Design first.** On a new app or a restyle, the agent writes `DESIGN.md` with five decisions and
   a reason for each. If your project already has a design system, it records that instead of
   inventing one: the project always wins.
2. **The always-on rules.** States, feedback, look, feel, words, safety, speed and finish: 21 short
   rules that apply to every screen.
3. **Depth on demand.** A routing table sends the agent to one reference file per kind of work. A
   form loads the forms guidance; a webhook loads the webhook guidance; nothing else enters context.
4. **A done check.** Before calling it done the agent runs the slop searches, walks each touched
   screen through its states, and tries cross-user access on every route. It tells you what it
   skipped.

## Boundaries

This skill does **not** cover:

- **A deep security audit of an existing backend.** It builds security in and runs a pre-ship
  checklist. For a full, finding-by-finding audit, use [backend-hardening](../backend-hardening).
- **Schema design and query tuning.** It covers pagination, N+1 and indexes at the level a feature
  needs. For a schema review or a new data model, use [database-design](../database-design).
- **Native mobile apps.** The rules target web apps, including mobile web. iOS and Android
  platform guidelines aren't covered yet.
- **Marketing pages as the main job.** Landing copy rules are included, but the focus is product UI.

## Files

```
not-a-demo/
├── SKILL.md                    # entry point: rule zero, DESIGN.md step, 21 rules, router, done check
├── SOURCES.md                  # where every rule comes from (never loaded by the agent)
└── references/                 # loaded only when the task needs it
    ├── design-decisions.md
    ├── visual-design.md
    ├── motion.md
    ├── ux-patterns.md
    ├── ux-writing.md
    ├── security.md
    ├── saas-essentials.md
    ├── frontend.md
    ├── backend.md
    ├── infra.md
    └── slop-check.md
```

Only `SKILL.md` enters context when the skill triggers. The references add depth without adding
cost to a small task.

## License

MIT
