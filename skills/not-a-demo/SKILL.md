---
name: not-a-demo
description: Makes AI-built web apps look designed and behave like shipped software instead of a demo. Covers a written design direction, loading/empty/error states, instant feedback, safe deletes, real UI copy, purposeful motion, server-side auth and data access, secrets, and speed from browser to database. Use when building, extending, restyling or reviewing any web app, SaaS, dashboard, admin panel, internal tool or app screen, or when the user says it looks AI-generated, generic, sloppy, unfinished, slow or insecure. For a full security audit of an existing backend use backend-hardening; for schema, index and migration review use database-design.
license: MIT
compatibility: Works in any agent that can read and edit project files. The optional checks in references/slop-check.md use ripgrep (rg) in bash or zsh.
metadata:
  version: 1.0.0
  author: nuu-maan
  homepage: https://qala.lol
---

# not-a-demo

A demo looks finished in a screenshot. A real app survives use: slow networks, empty accounts, wrong input, a phone, a keyboard, and a second user trying to read the first user's data. This skill closes that gap.

## Rule zero: fit the app

- **Always**: the rules below. Every app, every screen.
- **When it applies**: the screen has the thing (a list, a form, uploads, payments, an AI feature). Read the matching reference.
- **Only when asked or at scale**: growth features (referrals, onboarding questions, plan badges) and infra (Redis, load balancing, tracing). Suggest them; don't add them unasked. A todo app needs no audit log.
- **The project wins**: if the app already has a design system, tokens, components or conventions, use them. The user's explicit instructions beat every default here.

## Step 1: decide the design before writing UI

Before the first screen, write `DESIGN.md` at the project root with these decisions and a one-line reason for each. If it exists, follow it. If the project already has a theme, record that instead of inventing one. Details and defaults: [references/design-decisions.md](references/design-decisions.md).

1. **Theme**: light, dark or both.
2. **Accent**: one color, plus the neutral gray tinted toward it.
3. **Type**: the font and why it fits this product.
4. **Density**: compact (tools used all day) or comfortable (occasional use).
5. **Layout per screen type**: triage → list + detail, create → editor, set up → guided steps, decide → one focused view.

Check every screen against it. Ban lists alone breed new clichés; a written direction doesn't.

## Always

**States and feedback**
1. Every data view has loading (a skeleton shaped like the result), empty (what goes here + the button to create the first one), error (what failed + Retry) and loaded. Test with realistic data: 0, 1 and many rows, long names.
2. Every action responds within 100ms: an optimistic update or a pending button. Disable submit while pending.
3. Failures never lose what the user typed.
4. Reversible destructive actions happen at once with Undo. Permanent ones confirm by naming the item. High-stakes ones (project, workspace, account) make the user type the name.

**Look** ([visual-design.md](references/visual-design.md))
5. Every value comes from a scale: spacing in 4px steps, about 4 font sizes per screen, a small radius set, one shadow style. No one-off hex codes or pixel values.
6. One accent color, neutrals tinted toward it, never pure #000 or #fff. Text contrast at least 4.5:1.
7. One primary button per screen. Create emphasis by toning the rest down.

**Feel** ([motion.md](references/motion.md))
8. Animate only transform and opacity, under 300ms, ease-out. Never animate keyboard-triggered actions or things used 100+ times a day. Respect `prefers-reduced-motion`.
9. The keyboard works: a visible `:focus-visible` ring, Esc closes the top layer, dialogs trap focus and return it, Enter submits forms.
10. Works on a phone: reflows at 375px, tap targets at least 44px, inputs at least 16px, no page-level sideways scroll.
11. Browser behavior stays intact: real links (Cmd/Ctrl+click works), Back restores scroll, filters, tabs and pagination live in the URL, paste and zoom are never blocked.

**Words** ([ux-writing.md](references/ux-writing.md))
12. Buttons are verb + object ("Save changes", "Delete project"). Errors say how to fix it ("Enter an email like sam@acme.com"), never "Invalid" or "Oops". Sentence case. No hype words.

**Safety**, for any app with a server, login or user data ([security.md](references/security.md))
13. Check auth and ownership on the server in every route, API handler and server action, never only in middleware or the UI. Scope every query to the current user.
14. No secret reaches the browser. Anything prefixed `NEXT_PUBLIC_`, `VITE_`, `PUBLIC_`, `EXPO_PUBLIC_` or `REACT_APP_` is public. LLM keys, Stripe secret keys, Supabase `service_role` and database URLs stay on the server.
15. Validate every input on the server with a schema, write only allowlisted fields (never `update(req.body)`), parameterize every query.
16. Sessions live in `HttpOnly; Secure; SameSite` cookies, never localStorage. Rate-limit login, signup, password reset, one-time codes and AI endpoints.
17. Supabase: row-level security on every table. Firebase: owner-only rules. Never `using (true)` or `if true`.
18. Render user and model content as text. No `dangerouslySetInnerHTML`, `v-html` or `innerHTML` without DOMPurify.

**Speed** ([frontend.md](references/frontend.md), [backend.md](references/backend.md))
19. Images in WebP/AVIF with width and height, lazy below the fold. Self-hosted fonts, at most 3 weights. Independent requests run in parallel. Lists are paginated. Every external call has a timeout.

**Finish**
20. No dead ends: no "Coming soon" buttons, links to nowhere, lorem ipsum, John Doe or fake stats. If a feature can't ship with all its states, shrink it.
21. Before adding a dependency, confirm the package exists, the name is exact and it's maintained. Attackers register package names that AI invents.

## When it applies

| Building | Read |
|---|---|
| A new app, a new screen type or a restyle | [design-decisions.md](references/design-decisions.md), [visual-design.md](references/visual-design.md) |
| Animation, hover, menus, dialogs, toasts, drag, focus | [motion.md](references/motion.md) |
| Lists, tables, search, filters, bulk actions, forms, autosave, shortcuts, command palette, sidebar, plans, onboarding | [ux-patterns.md](references/ux-patterns.md) |
| Any user-facing text: buttons, errors, empty states, emails, landing copy | [ux-writing.md](references/ux-writing.md) |
| Login, accounts, settings, billing, email, dates, backups, launch metadata | [saas-essentials.md](references/saas-essentials.md) |
| A server, login, uploads, webhooks, URL fetching, AI features | [security.md](references/security.md) |
| Navigation speed, data fetching, forms, CSS, hydration, streaming | [frontend.md](references/frontend.md) |
| APIs, database, background jobs, money, caching, logging | [backend.md](references/backend.md) |
| Deploys, cache headers, monitoring, scaling | [infra.md](references/infra.md) |

For a full security audit of an existing backend, use the backend-hardening skill. For schema, index and migration review, use database-design.

## Never → instead

These are the most-reported AI tells. Use the replacement; don't just dodge the tell.

| Never | Instead |
|---|---|
| Untouched shadcn/Tailwind defaults | Your own accent, radius and font tokens, set before the first screen |
| Indigo/violet/purple accent, purple→blue gradients | The accent from DESIGN.md; gradients only as a subtle shift within one hue |
| Gradient text, glows, colored shadows | Solid text; emphasis by size or weight |
| Three identical icon-topped feature cards, four identical KPI cards | One dominant item, the rest smaller; icons beside headings, not in tiles above |
| Cards on everything, cards inside cards | Group with spacing, background steps and dividers |
| All-caps tracked label above every heading, "→" on every button, a badge above the H1 | A plain sentence-case heading |
| `border-l-4` accent stripe on cards | Nothing, unless it encodes status |
| Emoji as icons | One icon set, one stroke width |
| Fade-up on every section, `hover:scale` on cards, `transition-all`, pulsing dots | No motion, or one purposeful moment |
| "Unlock / Empower / Supercharge / Seamless", "It's not just X, it's Y" | What it does, concretely, with a number if you have one |
| Full-page spinner, "No data", "Something went wrong" | Skeleton; explanation + action; what failed + Retry |
| `alert()` / `confirm()` | Your own dialog or toast |
| Swapping one cliché for another (cream + serif + terracotta, black + acid green) | The direction in DESIGN.md |

## Before calling it done

1. Run the searches in [slop-check.md](references/slop-check.md) on the code you changed. Fix or justify every hit.
2. Walk every screen you touched: loading, empty, error, slow network (throttle to Slow 4G), 375px width, keyboard only, a double-clicked submit, very long text.
3. If there's a server: signed in as user A, request user B's records on every route. Each must return 403 or 404.
4. Fix what breaks. Tell the user what you skipped.
