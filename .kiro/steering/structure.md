# Structure

## Top-Level Layout

```
cc-lens/
  bin/cli.js          — npx entry point (plain JS, no transpile)
  app/                — Next.js App Router pages and API routes
  components/         — React components (feature + shared UI)
  lib/                — Server-side utilities and data layer
  types/claude.ts     — Shared TypeScript types for all Claude data shapes
  public/             — Static assets (screenshots, images)
```

## App Directory

Feature pages mirror the sidebar navigation. Each route is a directory with a `page.tsx`. Server components handle the shell; client components (suffixed `-client.tsx` or using `'use client'`) handle interactivity.

```
app/
  page.tsx                  — Overview (server shell → OverviewClient)
  overview-client.tsx       — Overview client component
  layout.tsx                — Root layout: Sidebar, BottomNav, ThemeProvider, KeyboardNavProvider
  globals.css               — Tailwind base + CSS custom properties (theme tokens)
  api/                      — Route handlers (one subdirectory per data domain)
    sessions/, projects/, costs/, tools/, activity/,
    history/, memory/, plans/, todos/, settings/,
    stats/, export/, import/
  sessions/
    page.tsx
    [id]/page.tsx            — Session detail / replay
  projects/
    page.tsx
    [slug]/page.tsx          — Project detail
  (other feature dirs follow the same pattern)
```

## Components Directory

Organised by feature, with shared pieces at the root and in `ui/`:

```
components/
  ui/                          — shadcn/ui primitives (badge, button, card, command, dialog, …)
  layout/                      — sidebar.tsx, bottom-nav.tsx, top-bar.tsx
  overview/, sessions/, costs/, tools/, activity/, projects/
                               — feature-specific components
  global-search.tsx            — ⌘K command palette
  keyboard-nav-provider.tsx    — Global keyboard navigation context
  theme-provider.tsx           — Dark/light theme context
  use-global-keyboard-nav.ts   — Hook for keyboard shortcuts
```

## Lib Directory

Pure utilities — no React, no Next.js imports:

```
lib/
  claude-reader.ts    — All fs reads from ~/.claude/; exports typed async functions
  decode.ts           — slug ↔ path conversion (slugToPath, projectDisplayName)
  pricing.ts          — Static model pricing table + cost calculation helpers
  replay-parser.ts    — Parses JSONL session lines into a replay-friendly structure
  tool-categories.ts  — Maps tool names to category labels (file-io, shell, agent, …)
  utils.ts            — cn() helper (clsx + tailwind-merge)
```

## Naming Conventions

- Feature pages: `app/<feature>/page.tsx` (server component, no directive)
- Client components: `'use client'` at top of file; often named `<Feature>Client` when paired with a server shell
- API routes: `app/api/<domain>/route.ts`; export named HTTP methods (`GET`, `POST`)
- Shared hooks: `use-<name>.ts` in `components/`
- shadcn components: match shadcn naming exactly (`card.tsx`, `badge.tsx`, etc.)

## Import Pattern

All cross-directory imports use the `@/` alias:
```ts
import { claudePath } from '@/lib/claude-reader'
import type { SessionMeta } from '@/types/claude'
import { cn } from '@/lib/utils'
```

Relative imports are only used within the same directory.
