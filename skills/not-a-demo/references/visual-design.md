# Visual design

## Color

- Build each color as a fixed scale where each step has one job (Radix 12-step):
  - 1-2: app background, subtle background
  - 3 / 4 / 5: component background at rest / hover / active or selected
  - 6: static borders and dividers. 7: interactive borders. 8: hover borders, focus rings
  - 9 / 10: solid fills (primary button) at rest / hover
  - 11: secondary text. 12: primary text
- Use Radix Colors, or define scales in OKLCH: vary lightness, keep chroma and hue steady. Don't invent shades on the fly with `lighten()`/`darken()`. Relative OKLCH (`oklch(from var(--accent) calc(l - 0.05) c h)`) is fine for small state changes.
- Darker shades: lower lightness, raise chroma slightly. Lighter: the reverse. Mixing in black makes mud.
- Neutrals plus one accent. Add color last, only where it means something: the primary action, selection, status.
- On a colored background, tint text, borders and shadows toward that hue. No grey text, no black shadows.
- Contrast: body text at least 4.5:1, large text and icons at least 3:1. Hover, active and focus states have more contrast than rest.
- Status colors: red error, green success, amber warning, blue info. Never color alone: pair it with an icon or text.

## Hierarchy

- Only the page title gets every emphasis lever (size + weight + color). Everything else trades one off.
- Emphasize by de-emphasizing: make secondary text smaller or lower-contrast instead of making primary text bolder.
- Labels are a last resort: "12 left in stock" beats "Stock: 12".
- Navigation chrome recedes: sidebar dimmer than content, small icons, few dividers, no colored icon tiles.
- One primary button per view; the rest secondary or ghost.

## Spacing and layout

- Every spacing value comes from the scale: 4, 8, 12, 16, 24, 32, 48, 64, 96. Nothing in between.
- Tight inside groups, loose between them. Outer padding at least as big as inner padding. Headings get more space above than below.
- Start with too much whitespace, then remove some. Typical: panels and sidebars 16px, cards 20-24px, page edges 24-32px.
- Buttons: horizontal padding about twice the vertical (`px-4 py-2`).
- Align everything to something. Nudge icons by 1px when optical alignment beats geometric.

## Typography

- About 4 sizes per screen: title, body, secondary (about 2px smaller), tertiary.
- Line height 1.5 for body, tighter as size grows: about 1.25 at 24px, 1.1 at 48px and up.
- Letter-spacing: tighten large text (about -0.01em at 20px, -0.02em at 24px and up), leave body near 0, add +0.05em to +0.1em to all-caps.
- Line length 45-75 characters: `max-width: 65ch`.
- App UI weights 400 and 500; 600 sparingly. Nothing under 400.
- `font-variant-numeric: tabular-nums` on numbers that change or line up. `text-wrap: balance` on headings, `text-wrap: pretty` on paragraphs. Real "…" and curly quotes.

## Depth

- Fewer borders. Separate with spacing, a background step or a shadow. Never two hard dividers next to each other. Tables: drop vertical borders, add padding, keep light row dividers.
- One depth style throughout: hairline borders or soft shadows. Floating layers (menus, popovers) may combine a faint border with a shadow.
- Shadows: one light source from above, two or more layers (ambient + direct), blur about twice the y-offset, low opacity. Higher elevation means bigger offset and blur, lower opacity. Example: `0 1px 2px rgb(0 0 0 / .06), 0 4px 12px rgb(0 0 0 / .06)`.
- Radius: a small set tied to size (controls 6px, cards/menus/modals 12px). Nested corners: inner radius = outer radius - gap.

## Gradients

- Only a subtle shift within one hue. Add grain so it doesn't band or look cheap: SVG `feTurbulence` noise on a `::after`, blended with `overlay` at 15-50% opacity.

```css
.gradient { position: relative; isolation: isolate; }
.gradient::after {
  content: ""; position: absolute; inset: 0; z-index: -1; pointer-events: none;
  mix-blend-mode: overlay; opacity: .25;
  background: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='200' height='200'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='.8' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
}
```

- Grain should be felt, not seen. Visible noise, dithered pixel patterns and blurred color blobs are the opposite: decoration that reads as AI.

## Dark mode

- Show elevation with lighter surfaces, not shadows. The base isn't pure black: tint it toward the accent, e.g. `oklch(16% 0.01 <accent hue>)`.
- Dark colors are their own palette, not inversions. Swap tokens; don't rewrite components.
- Avoid saturated grays as the app background. Body text still needs 4.5:1 (zinc-400 on near-black often fails).
- Set `color-scheme: dark` so scrollbars and native controls match.

## Icons and imagery

- One icon set, one stroke width. 16px inline, 20px standalone, an 8px gap to labels, slightly lower contrast than the text next to them.
- Icons only where they help scanning. Not sparkles, zap, rocket, shield-check and bot as decoration.
- Text over images needs a treatment: a ~35% black overlay, a bottom gradient or a scrim.

## Data-heavy screens

- Density is value per pixel, not crowding. Same role = same size, font and color.
- KPI tiles: label + number + comparison ("+12% vs last month") + time range. One hero metric larger than the rest. No decorative sparklines.
- Table numbers right-aligned and tabular. Long text truncates with the full value on hover.
