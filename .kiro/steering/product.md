# Product

## Purpose

cc-lens (Claude Code Lens) is a local analytics dashboard for Claude Code users. It reads directly from `~/.claude/` and presents usage data, costs, sessions, and activity in a browser UI — no cloud, no telemetry, no authentication.

## Core Value

Users get full visibility into their Claude Code usage without sending data anywhere. Everything runs locally: the Next.js app starts on a free port, opens in the browser, and refreshes every 5 seconds from local files.

## Distribution Model

Published to npm as `cc-lens`. The primary entry point is `npx cc-lens` — no install required. The CLI copies the Next.js app into `~/.cc-lens/`, installs dependencies there, and launches the dev server.

## Feature Areas

**Analytics**: Token usage over time, cost by model and project, cache efficiency, peak-hour heatmaps, activity streaks.

**Session Inspection**: Sortable/filterable session list with emoji badges (compacted, agent, MCP, web, thinking); full session replay with per-turn token display and timeline chart.

**Project View**: Card grid with per-project stats; detail page with cost chart, most-used tools, git branches.

**Claude Code Data Browser**: History (history.jsonl), Todos, Plans, Memory files, Settings, Skills, Plugins — read-only inspection of the full `~/.claude/` directory.

**Navigation**: Global search via `⌘K`/`/`; `j`/`k` keyboard navigation in session list; `g`+letter shortcuts to jump between pages.

**Export/Import**: Download `.ccboard.json` or `.zip` with full JSONL; additive merge preview on import.

## Data Sources

All data comes from `~/.claude/`:
- `projects/<slug>/*.jsonl` — primary session data (JSONL lines)
- `stats-cache.json` — aggregated stats (daily model tokens used for cost chart)
- `usage-data/session-meta/` — fallback if JSONL is absent
- `history.jsonl`, `todos/`, `plans/`, `memory/`, `settings.json`, `skills/`, `plugins/`
