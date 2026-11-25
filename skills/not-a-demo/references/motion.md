# Motion and interaction

## Whether to animate

- Never animate keyboard-triggered actions or anything used 100+ times a day: the command palette, context menus, list navigation, adding or removing list items.
- Used tens of times a day: shorten it or remove it. Save delight for rare moments (first run, finishing something big).
- At most one orchestrated moment per page. No fade-up on every section.

## Timing

| Element | Duration |
|---|---|
| Press feedback | 100-160ms |
| Tooltip, small popover | 125-200ms |
| Dropdown, select | 150-250ms |
| Modal, drawer | 200-300ms (up to 500ms for full-height sheets) |

- Exits about 20% faster than entrances (250ms in, 200ms out).
- Keep UI motion under 300ms.

## Easing

- Entering and exiting: ease-out, e.g. `cubic-bezier(0.23, 1, 0.32, 1)`. The built-in CSS `ease-out` is often too weak.
- Moving or morphing on screen: ease-in-out, e.g. `cubic-bezier(0.77, 0, 0.175, 1)`.
- Hover and color changes: `ease`. Constant motion (spinners): `linear`.
- Never ease-in for anything entering.
- Springs only for movement and gestures, never for color or opacity. Start with no bounce; add 0.1-0.3 bounce only after a flick with momentum.

## How to animate

- Animate only `transform` and `opacity`. Never `transition: all`; list the properties. Never animate width, height, top, left, margin or padding.
- Enter from `scale(0.95)` + `opacity: 0` (dialogs `0.97`), never `scale(0)`. Press: `scale(0.97)` on `:active`.
- Popovers and menus grow from their trigger: set `transform-origin` to it (Radix: `var(--radix-popover-content-transform-origin)`). Modals grow from the center.
- Use CSS transitions for anything that can be toggled mid-way; they retarget smoothly, while keyframes restart from zero.
- Under `prefers-reduced-motion: reduce`, remove movement and keep short opacity or color fades.
- Switch themes instantly: disable transitions while the theme changes, so colors don't fade across the page.
- Steps in a flow slide in the direction of travel: Next brings the new step in from the right, Back from the left. A cross-fade hides which way you went.
- Height to `auto` (accordions, disclosures): `:root { interpolate-size: allow-keywords }` where supported; elsewhere it snaps, which is fine. This is the one exception to never animating height; keep it under 250ms.
- Stagger list items 30-50ms apart, cap the total, and never block input while it runs.

## Hover, press, focus

- Wrap hover styles in `@media (hover: hover) and (pointer: fine)`, or touch devices get stuck hover states.
- Everything pressable has an `:active` state. Highlight on pointer-down; fire on release, so dragging off cancels.
- Hover never changes size or font weight (layout shift). Change color, background or shadow instead.
- Use `:focus-visible`, not `:focus`. The ring is at least 2px with 3:1 contrast. Sticky headers must not cover the focused element: use `scroll-padding-top`.

## Menus, tooltips, dialogs, toasts

- **Use a headless library** (Radix, React Aria, Base UI, Headless UI, Ark) for menus, dialogs, comboboxes, popovers and tooltips. Hand-rolled ones miss keyboard and focus behavior.
- **Tooltips**: a 500-700ms delay on first hover, instant for the next one within about 300ms. They also open on focus and close on Esc. No interactive content inside. Never the only explanation for a disabled button.
- **Menus**: Enter, Space or ↓ opens with focus on the first item. Arrows move, Home/End jump, typing a letter jumps to a match, Esc closes and returns focus to the trigger. Submenus need a tolerance zone so diagonal mouse moves don't close them.
- **Dialogs**: focus the first field, or the safe button in destructive dialogs. Tab wraps inside, Esc closes, focus returns to the opener. Lock background scroll without a layout jump (`scrollbar-gutter: stable`, `overscroll-behavior: contain`).
- **Toasts**: auto-dismiss after 4-6s; pause on hover and while the tab is hidden. Undo toasts stay 8-10s. Never the only place important information or an action appears.

## Gestures, drag, hit areas

- Drag follows the pointer from the grab point, starts after about 10px of movement, and snaps or dismisses based on velocity. Destructive actions commit only on release.
- Springs take the release velocity, so a flicked sheet keeps the finger's speed instead of restarting from zero (Motion and most gesture libraries pass it through).
- Dragging past an edge or limit meets resistance: move by a shrinking fraction of the overshoot (e.g. `limit + overshoot * 0.2`), then spring back on release. Never a hard stop.
- Drag and drop: `cursor: grab`, the original dimmed to about 40% opacity, a 2px drop line, and a keyboard "Move to…" alternative.
- Targets at least 24px on desktop, 44px on touch. Grow small icon buttons with padding or a pseudo-element. No dead gaps between list items (padding, not margin). A checkbox and its label share one target.
- `touch-action: manipulation` on controls. `user-select: none` on control labels only, never on content.

## Feedback details

- Copy buttons swap to a check icon in place for 1.5-2s.
- A failed submit (wrong password, invalid code) can shake the field: a few px side to side, about 300ms, next to the error text, never instead of it. None under reduced motion.
- Loading indicators appear after 150-300ms; once shown, they stay at least 300-500ms so they don't flash. Loading buttons keep their label and add a spinner.
- Numbers that change (timers, counters, prices) use `tabular-nums` so they don't jitter.
