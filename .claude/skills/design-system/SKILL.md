---
name: design-system
description: Theming and visual-consistency rules — reuse existing design tokens (CSS variables, Tailwind theme, theme.ts, component library theme) for color, spacing, typography, radius, shadow and breakpoints; no hardcoded colors; semantic tokens; dark mode; accessible contrast and focus states. Use whenever writing or changing UI styles, adding colors, building UI primitives, or setting up a theme.
---

# Design System / Theming

## Purpose

One visual language, controlled from one place. Changing `--primary` should recolor the app; no component should need editing to follow a brand change.

## Rules

### R1. Find the token source before styling
Look in this order: `globals.css` / `index.css` (`:root` CSS variables), `tailwind.config.*` (`theme.extend`) or Tailwind v4 `@theme` in CSS, `theme.ts` (MUI/Chakra/styled-components), `components.json` (shadcn/ui), design-token JSON. Use what exists. If nothing exists and the task touches more than a couple of components, propose a minimal token set (R6) before adding styles.

### R2. No hardcoded project-wide values in components
- ❌ `text-[#10B981]`, `bg-emerald-500` for the brand color, `style={{ color: '#333' }}`, `margin: 13px`.
- ✅ `text-primary`, `bg-background`, `var(--primary)`, `theme.palette.primary.main`, spacing scale (`p-4`, `gap-6`).
- Arbitrary values are acceptable only for genuinely one-off layout needs (e.g., a specific illustration offset) — never for colors, font sizes, radii or shadows that should be consistent.

### R3. Semantic tokens over raw palette
Components use **role** tokens: `primary`, `primary-foreground`, `secondary`, `muted`, `muted-foreground`, `accent`, `destructive`, `success`, `warning`, `border`, `input`, `ring`, `background`, `foreground`, `card`, `popover`. Raw palette values (`emerald-500`) are referenced only inside the token definitions.

### R4. Status colors via mapping, not inline
```ts
// constants/statuses.ts
export const STUDENT_STATUS_BADGE: Record<StudentStatus, BadgeVariant> = {
  active: 'success', suspended: 'warning', graduated: 'secondary',
};
```
The `Badge` primitive maps variants → tokens. Components never pick colors for statuses.

### R5. Reuse primitives
Before writing a styled button/input/modal/table, check `components/ui`. Extend a primitive with a variant (`cva`/`variants`) rather than creating `PrimaryButton2`. Don't override primitive styles ad hoc in features; add a variant if the need recurs.

### R6. Minimal token set (when creating one)
```css
:root {
  --background: 0 0% 100%;
  --foreground: 222 47% 11%;
  --primary: 160 84% 39%;            /* #10B981 */
  --primary-foreground: 0 0% 100%;
  --muted: 210 40% 96%;
  --muted-foreground: 215 16% 47%;
  --border: 214 32% 91%;
  --ring: 160 84% 39%;
  --destructive: 0 84% 60%;
  --success: 142 71% 45%;
  --warning: 38 92% 50%;
  --radius: 0.5rem;
  --font-sans: var(--font-inter), system-ui, sans-serif;
}
.dark {
  --background: 222 47% 11%;
  --foreground: 210 40% 98%;
  /* …override only what changes */
}
```
Map them in Tailwind (`colors: { primary: 'hsl(var(--primary) / <alpha-value>)' }`) or `@theme` in Tailwind v4. Match the color format the project already uses (hex, HSL channels, OKLCH).

### R7. Scales, not values
- Spacing: use the scale (Tailwind default 4px steps or the theme's `spacing()`).
- Typography: a fixed type scale (`text-sm`, `text-base`, `text-lg`, `text-2xl`…) + weights limited to those loaded.
- Radius: `rounded-[var(--radius)]` / `rounded-md` derived from `--radius`.
- Shadows: 2–3 named levels.
- Breakpoints: the configured ones only; no custom media queries with magic widths.

### R8. Dark mode
If the project supports dark mode, every new color must come from a token that has a dark value. Test both themes. Don't use `bg-white`/`text-black` for surfaces/text.

### R9. Accessibility is part of the design system
- Text contrast ≥ 4.5:1 (3:1 for large text/UI icons). Check when adding a token pairing.
- Visible focus state on all interactive elements (`focus-visible:ring-2 ring-ring`); never `outline: none` without a replacement.
- Don't convey status by color alone — add text/icon.
- Use semantic HTML (`button`, `a`, `label`) before ARIA.

## Implementation Guidelines

1. Locate token source and primitives.
2. Build UI with tokens and primitives only.
3. If a needed token is missing, add it centrally (light + dark), then use it.
4. Grep your diff for `#`, `rgb(`, `[`-arbitrary Tailwind values and raw palette classes; replace with tokens.

## Examples

```tsx
// ❌
<button className="bg-[#10B981] hover:bg-[#0e9f6e] text-white px-[14px] py-[9px] rounded-[7px]">Save</button>

// ✅
<Button variant="default" size="md">Save</Button>
// Button internally: "bg-primary text-primary-foreground hover:bg-primary/90 h-10 px-4 rounded-md"
```

## Anti-patterns

- Brand hex values copy-pasted into many components.
- Using raw palette classes (`bg-emerald-500`) for brand/semantic meaning.
- Five slightly different grays and border radii across screens.
- New one-off button components instead of variants.
- Adding a second UI kit for one component.
- Removing focus outlines.

## Checklist

- [ ] Existing token source identified and used.
- [ ] No hardcoded colors/sizes/radii/shadows in components (diff grep clean).
- [ ] Semantic tokens used; status colors via mapping.
- [ ] Primitives reused or extended with variants.
- [ ] Dark mode (if supported) verified.
- [ ] Contrast and focus-visible states OK.
