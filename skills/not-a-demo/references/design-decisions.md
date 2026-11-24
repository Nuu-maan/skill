# Design decisions

Write these to `DESIGN.md` before building UI. Keep it under 30 lines. Every screen gets checked against it.

## Template

```md
# Design
Product: <who uses it, how often, for what job>
Theme: light | dark | both. Why:
Accent: <oklch or hex>. Neutral: <gray tinted toward the accent>. Why:
Type: <font> (+ <second font> only for a specific job). Why:
Density: compact | comfortable. Why:
Screens: <screen> → <layout>   e.g. Inbox → list + detail drawer
Radius: controls 6px, cards/menus/modals 12px
Depth: hairline borders | soft layered shadows
Terms: <one word per concept, e.g. "project" not "workspace">
Signature: <the one memorable detail; spend boldness here and nowhere else>
```

## How to choose

- **Product first.** Who uses it, how often, for what job. A tool used all day gets compact density, quiet chrome and almost no motion. Something used monthly gets comfortable spacing and more guidance.
- **Theme.** Dark is a decision, not a fallback. Most business apps default to light; dark suits media, code and night-time use.
- **Accent.** One hue, from the brand if there is one. Avoid stock Tailwind indigo/violet (`#6366f1`, `#8b5cf6`, `#7c3aed`), the most common AI default.
- **Neutral.** A gray carrying a little of the accent's hue. Radix pairs: slate with blue, mauve with purple or pink, sage with green or teal, olive with lime, sand with yellow, orange or brown. Warm or cool, never both.
- **Type.** Pick the body font first. Add a second only to fill a gap (a mono for code, a serif for long reading). Never Thin or Light weights in UI. A default font like Inter is fine when the reason is written down.
- **Screens.** Pick layout by the screen's job, not one shell (sidebar + 4 KPI cards + chart + table) for everything:
  - triage or review → list + detail
  - create → editor or canvas
  - set up → guided steps
  - decide → one focused view with the key number
  - monitor → dashboard with one hero metric
  - settings → grouped sections with clear labels, not a "Danger Zone" card collage
- **Signature.** One thing that makes this product recognizable: a distinctive accent, an illustration style, a beautifully dense table. Only one.

## Defaults when the user gives no brand

- **Font**: Inter (Inter Display for large headings) or Geist. Alternatives: Figtree, Satoshi, Source Sans 3. `tabular-nums` in data.
- **Type scale**: 12 / 14 / 16 / 18 / 20 / 24 / 30 / 36px. App body 14px (13px minimum for dense tools); reading pages 16-18px.
- **Spacing**: 4px base: 4, 8, 12, 16, 24, 32, 48, 64, 96.
- **Radius**: controls 6px, cards/menus/modals 12px, full-screen sheets 16px.
- **Color**: 12-step scales built in OKLCH or taken from Radix Colors (step roles in [visual-design.md](visual-design.md)). Semantic: red error, green success, amber warning, blue info.
- **Borders**: 1px, neutral step 6 for static dividers, step 7 for interactive edges.
- **Icons**: Lucide (or the project's set) at 16px inline, 20px standalone, one stroke width.

## Don't swap one cliché for another

Anti-slop prompts produce their own look. Avoid these unless the brand asks for them:
- warm cream background + serif display + terracotta accent
- near-black background + acid-green accent
- one italic serif word inside a sans-serif headline
- zero-radius "editorial broadsheet" layouts
- fonts chosen only to avoid Inter (Space Grotesk, Instrument Serif) with no other reason
