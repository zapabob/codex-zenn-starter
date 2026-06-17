# AGENTS.md

## Project
Next.js application with npm. App Router under `src/app/`.

## Commands
- Install: `npm ci`
- Dev: `npm run dev`
- Test: `npm test`
- Lint: `npm run lint`
- Typecheck: `npm run typecheck`
- Build: `npm run build`

## Rules
- Do not commit `.env.local` or any file containing API keys.
- Do not edit generated output under `.next/`.
- Prefer existing components in `src/components/`.
- Use Server Components by default; add `"use client"` only when needed.

## Done
- `npm test` passes.
- `npm run lint` and `npm run typecheck` pass.
- No new TypeScript errors in changed files.
