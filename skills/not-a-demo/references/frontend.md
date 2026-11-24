# Frontend

Targets: LCP under 2.5s, INP under 200ms, CLS under 0.1, at the 75th percentile on mobile. Interactions under 100ms, ideally 50ms. Measure before optimizing.

## Navigation feels instant

- Prefetch on intent: React Router/Remix `<Link prefetch="intent">`, TanStack Router `defaultPreload: 'intent'`, Next.js `<Link>` (prefetches in the viewport). Multi-page apps: Speculation Rules with `"eagerness": "moderate"`. Exclude logout and other side-effect URLs.
- Every dynamic route has an instant shell: Next.js `loading.tsx`, a Suspense boundary around the component that fetches, not around the whole layout.
- One global navigation progress bar, delayed about 150ms.
- No waterfalls: start independent fetches together (route loaders, server components, `Promise.all`), not each component fetching after its parent renders.
- Detail views start from list data, so opening an item doesn't flash a blank loading state (TanStack Query `initialData` / `placeholderData`).

## Pending and optimistic

- Optimistic for predictable single-field updates. A pending indicator for creates, URL changes, payments and email sends. Never optimistic for payments or logout.
- Last request wins: abort superseded requests (AbortController) and ignore responses older than the current one.
- After a mutation, revalidate or invalidate queries instead of hand-patching state in many places.

## Back, history, URL

- Keep pages eligible for the back/forward cache: no `unload` listeners (use `pagehide`), `beforeunload` only while there are unsaved changes, no `Cache-Control: no-store` on non-sensitive HTML. On `pageshow` with `event.persisted`, refresh stale data.
- Back restores the scroll position (React Router `<ScrollRestoration/>`; Next.js does it by default). `scroll-padding-top` for sticky headers.
- The URL is state: filters, sort, tabs, pagination, the selected item, the open panel. Push history for changes worth going Back to; replace for minor ones. Search is a GET form.
- Navigation uses real `<a href>` so Cmd/Ctrl+click and middle-click open new tabs.

## Data fetching

- With TanStack Query or similar: set `staleTime` on purpose (30s to minutes); the default refetches on every mount and window focus.
- Never copy query data into `useState`; it stops background updates.
- Every variable a query uses goes in its key.

## Deploys, sessions, offline

- Tabs left open across a deploy request deleted chunks. Handle `vite:preloadError` / chunk-load errors by reloading, but only when no input is unsaved. HTML is `no-cache`; hashed assets are immutable.
- When the session expires, prompt to sign in again and keep the page state. Sync logout across tabs (BroadcastChannel or the `storage` event).
- Offline: a non-blocking banner, network-only actions disabled, edits queued with "Saved, will sync", and a last-updated time. Never auto-refresh content the user is reading.

## Forms

- A real `<form>` with `<label for>`, the correct `type`, `name` and `autocomplete` (`email`, `username`, `current-password`, `new-password`, `one-time-code`, `street-address`…). Enter submits.
- Works before hydration where the framework allows it (server actions, Remix actions). Wrap fields in `<fieldset disabled>` while submitting.
- Submit stays enabled until submission starts. On error, focus the first invalid field.
- Inputs at least 16px on mobile so iOS doesn't zoom. Never block paste.

## Layout shift and CSS baseline

- Images: `width` + `height` attributes with `height: auto`, or `aspect-ratio` for fixed crops. Reserve space for late content with `min-height`.
- Fonts: self-hosted woff2, at most 3 weights or one variable font, `font-display: swap`, preload the main one, and match fallback metrics (`size-adjust`; `next/font` does this for you).
- `scrollbar-gutter: stable` on scroll containers. `100dvh` or `100svh` instead of `100vh`. `overscroll-behavior: contain` on modals, drawers and chat panes.
- Baseline reset: `input, button, textarea, select { font: inherit }`, `overflow-wrap: break-word`, `text-wrap: balance` on headings and `pretty` on paragraphs, `isolation: isolate` on the app root, `touch-action: manipulation` on controls, and `color-scheme` set.
- Container queries (`container-type: inline-size`) for components that change with their own width; media queries for page-level layout.

## Images and scripts

- WebP/AVIF through the framework's image component or `<picture>` with `srcset`/`sizes`. `loading="lazy"` below the fold. The hero (LCP) image gets `fetchpriority="high"` and is never lazy.
- Rely on the framework's per-route code splitting. Lazy-import only heavy, rarely used pieces: rich-text editors, charts, maps, PDF viewers.
- Chat widgets, analytics and support tools load after the page, on idle or first interaction.

## Long content

- Virtualize lists over about 200 rows (TanStack Virtual, react-window, or whatever the project already uses), or paginate.
- `content-visibility: auto; contain-intrinsic-size: auto 500px` on long off-screen sections.

## React correctness

- In SSR apps, don't read `window`, `localStorage`, `matchMedia`, `Date.now()` or random values during render; read them after mount, or the hydrated page won't match the server's.
- Before reaching for `memo`: move state down, or pass components as `children`. Then profile, and memoize only what the profiler shows.
- Error boundaries per route section with a "Try again" reset, so one widget crashing doesn't blank the page. They don't catch event-handler or async errors; handle those explicitly.

## Streaming AI responses

- Stream tokens as they arrive. Show a Stop button. Render markdown incrementally, sanitized ([security.md](security.md)). Keep scroll pinned to the bottom unless the user scrolled up. If the stream fails, keep the partial output and offer Retry.

## View transitions (optional polish)

- Multi-page apps: `@view-transition { navigation: auto; }`. SPAs: feature-detect `document.startViewTransition`. Fetch data before starting the transition. Every `view-transition-name` must be unique. Turn them off under reduced motion, and keep them subtle because navigation is frequent.

## Tests that matter

- Mostly integration tests of real user flows. Query by role (`getByRole`), drive with `user-event`, wait with `findBy*`. Mock as little as possible.
