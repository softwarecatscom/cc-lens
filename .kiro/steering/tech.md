# Tech

## Stack

- **Framework**: Next.js 16 (App Router) with Turbopack; `turbopack.root` explicitly set to prevent npx workspace detection issues
- **Runtime**: React 19, TypeScript 5 (strict)
- **Styling**: Tailwind CSS v4 with `@tailwindcss/postcss`; utility-first, no CSS modules
- **UI Components**: shadcn/ui pattern — Radix UI primitives wrapped in `components/ui/`; `class-variance-authority` + `clsx` + `tailwind-merge` for variant composition
- **Charts**: Recharts
- **Command Palette**: cmdk (`CommandDialog` from shadcn)
- **Data Fetching**: SWR for client-side polling; Next.js Route Handlers for all data APIs
- **Date Utilities**: date-fns v4

## Key Architectural Decisions

**Local file I/O lives in `lib/claude-reader.ts`**: All reads from `~/.claude/` go through this module. Route handlers call it server-side; no direct fs calls in components.

**Route Handlers, not server actions**: All data is exposed via `app/api/**` as REST endpoints. Client components fetch with SWR using these routes. This keeps the data layer testable and explicit.

**Primary data source is JSONL**: `getSessions()` prefers `projects/<slug>/*.jsonl` over `usage-data/session-meta/` (fallback). Session metadata is derived by parsing JSONL lines at read time — no pre-computed index.

**Pricing is static in code**: `lib/pricing.ts` holds per-model input/output/cache rates. Fuzzy prefix matching handles model name variants (e.g., `claude-opus-4-5-20251101` matches `claude-opus-4-5`).

**npx distribution via cache dir**: CLI copies source files into `~/.cc-lens/` and runs `npm install` there on first run (or version change). This avoids Turbopack workspace root violations when invoked via npx.

## Conventions

- All imports use `@/` alias (maps to repo root via `tsconfig.json` `paths`)
- Client components are explicitly marked `'use client'`; server components have no directive
- Error handling in data layer: all async functions return `null` or `[]` on error, never throw
- JSONL parsing: always split on `/\r?\n/`, skip empty lines, wrap each `JSON.parse` in try/catch
- Fonts: `Geist Mono` for body, `Press Start 2P` for the wordmark
- Theme: dark/light toggle persisted in `localStorage`; `dark` class on `<html>`; default is dark

## Node Requirements

Node.js 18+ (uses `fs/promises`, top-level async in CLI).
