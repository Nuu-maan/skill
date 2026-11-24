# UX patterns

Each pattern: when to use it, how real apps do it, and the usual AI mistake.

## Optimistic UI updates
- **When**: predictable single-field changes: likes, toggles, renames, reordering, checking off items.
- **Not for**: creating records (no ID yet), anything that changes the URL, payments, sending email, logout. Show a pending state for those.
- **Do**: update instantly ("42 likes"), sync in the background, show a quiet "Syncing…" only if it takes over ~1s. On failure, roll back and show an error with Retry.
- **Mistake**: no rollback, so the UI lies after a failed request.

## Skeleton loaders
- **Do**: grey blocks shaped like the real content (same heights, rows, avatar circles) so nothing jumps when data arrives. Wait ~200ms before showing one; once shown, keep it at least ~300ms so it doesn't flash.
- **Mistake**: one generic grey box, or a centered spinner over a blank page.

## Command palette
- **When**: apps with more than a handful of pages or actions.
- **Do**: Cmd+K on Mac, Ctrl+K elsewhere. Fuzzy search over pages, actions and recent items. Arrow keys move, Enter runs, Esc closes. Show each action's shortcut next to it. No open/close animation.
- **Mistake**: a palette that only searches page names.

## Persistent sidebar navigation
- **Do**: highlight the active item, keep scroll and collapse state across pages, collapsible on desktop, a drawer or bottom nav on mobile. Dimmer than the content; no colored icon tiles.
- **Mistake**: the sidebar reloads or loses state on every navigation.

## Loading and empty states
- **Do**: the empty state says what lives here and gives the primary action: "No projects yet" + "New project". Tell "nothing yet" apart from "no results for this filter"; the second offers "Clear filters".
- **Mistake**: the same empty message for both, or no button.

## Autosave
- **Do**: save about 1s after typing stops and on blur. Status reads "Saving…" → "Saved" → "Couldn't save. Retry". Warn before leaving with unsaved changes (`beforeunload` only while there are any). Never silently overwrite a newer server version: detect the conflict and ask.
- **Mistake**: saving on every keystroke, or no visible status.

## Debouncing
- **When**: search, autocomplete, live filters, autosave.
- **Do**: 250-300ms for search. Cancel stale requests (AbortController) so an old response never overwrites a newer one. Five keystrokes should send one request.
- **Mistake**: debouncing without cancelling, so results flicker out of order.

## Throttling
- **When**: scroll, resize, mousemove, drag, infinite-scroll triggers.
- **Do**: run at most every ~100ms, or once per animation frame for visual updates.

## Smart form validation
- **Do**: validate on blur first, then live once a field shows an error so it clears the moment it's fixed. Say how to fix it ("Enter an email like sam@acme.com"). On submit, focus the first invalid field. Show server errors on the right field. Use the right `type` and `autocomplete` (`email`, `current-password`, `new-password`, `one-time-code`). A show-password toggle instead of a confirm-password field. Never block paste.
- **Mistake**: red errors while the user is still typing for the first time, or clearing the form on error.

## Bulk actions on lists
- **Do**: a checkbox per row and a select-all in the header (offer "Select all 248" when only the page is selected). Shift-click selects a range. An action bar shows "3 selected" with the actions. Destructive bulk actions confirm with the count.
- **Mistake**: no way to clear the selection, or selection lost on pagination.

## Reusable data table
- **Do**: one shared component with sorting, pagination or virtualization, a sticky header, right-aligned `tabular-nums` numbers, truncated long text with the full value on hover, and loading, empty and error states built in. Row click opens; row actions sit in a menu.
- **Mistake**: every page hand-rolls its own table with different behavior.

## Advanced data filtering
- **Do**: active filters as removable chips plus "Clear all". Keep filters in the URL query so they survive refresh and can be shared; push history for filter changes, replace for keystroke-level updates. Show the result count.
- **Mistake**: filters reset on refresh or on the back button.

## Keyboard shortcuts
- **Do**: Cmd/Ctrl+S saves, Esc closes the top layer, `/` focuses search, `?` lists all shortcuts. Show shortcuts in tooltips and menus. Don't fire single-key shortcuts while the user types in an input. Don't override browser shortcuts.
- **Mistake**: shortcuts nobody can discover.

## Tier-based UI indicators
- **When**: only if the app has plans.
- **Do**: usage like "Projects 3/10" with a bar, a warning from 80%. Locked features show a Pro badge and an upgrade path instead of being hidden or failing with an error.

## Onboarding data capture
- **When**: only if asked.
- **Do**: 1-3 questions ("What's your role?"), skippable, and each answer changes something (defaults, templates, what shows first).
- **Mistake**: a long form whose answers change nothing.

## Referral and affiliate pages
- **When**: only if asked.
- **Do**: the reward for both sides in one line ("Give $20, get $20"), a copy-link button that shows "Copied", and the status of each referral.

## Custom 404 page
- **Do**: match the app's look, say what happened in plain words, link home and to search. Make a matching 500 page.

## Mobile-optimized layouts
- **Do**: tables become cards or scroll inside their own container. The sidebar becomes a drawer. Dialogs become bottom sheets. Inputs use at least 16px text so iOS doesn't zoom. Use `100dvh`, not `100vh`. Respect safe-area insets. Primary actions within thumb reach.
- **Mistake**: a desktop layout squeezed to 375px.

## Destructive action protection
- **Do**, by stakes: reversible → act now with an Undo toast (8-10s). Permanent → a dialog naming the item ("Delete 'Q3 Report'?"). High stakes (project, workspace, account) → type the name; the button stays disabled until it matches. The button names the action ("Delete project"), never "OK". Prefer soft delete with Undo.

## Graceful error recovery
- **Do**: say what failed in plain words ("Couldn't save changes"), keep the user's input, give a Retry button. Auto-retry idempotent reads with backoff. Show a banner when offline. When the session expires, ask the user to sign in again without losing the page. Use error boundaries so one broken widget doesn't blank the page.
- **Mistake**: a raw stack trace or a blank screen.
