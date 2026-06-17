# DESIGN.md

## Stack
- Framework: Next.js (App Router)
- Styling: Tailwind CSS
- Components: shadcn/ui pattern under `components/ui/`

## Tokens
- Use CSS variables in `globals.css` for colors and radius.
- Do not hardcode hex values in components; use theme tokens.

## Layout
- Mobile-first responsive layout.
- Max content width: 72rem for article-style pages.

## Components
- Reuse existing UI primitives before adding new ones.
- Spacing: Tailwind scale only (no arbitrary pixel soup).

## Accessibility
- Keyboard focus visible on interactive elements.
- Images require meaningful `alt` text.

## Copy
- User-facing strings: clear Japanese or English per project locale.
- Chart captions in English when using data visualization tools.
