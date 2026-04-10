# CC-Lens Dashboard — UI Requirements

> **Scope**: Frontend / dashboard only. Backend implementation is out of scope for this document.
> **Data source**: All data is fetched from the backend API (see `cc-lens-dashboard-api-contract.md`).
> **Discovery method**: Systematic Chrome inspection of the running application at `localhost:3000`, correlated with source code in this repository.

---

## 1. Global Shell

### 1.1 Layout

- THE SYSTEM SHALL render a full-height flex layout consisting of a fixed left sidebar (desktop) and a main content area.
- THE SYSTEM SHALL render a fixed bottom navigation bar on mobile viewports (below `md` breakpoint).
- THE SYSTEM SHALL apply the `Geist Mono` font family as the primary monospace font and `Press Start 2P` as the display/accent font throughout the interface.
- THE SYSTEM SHALL persist the user's selected theme (`dark` or `light`) in `localStorage` under the key `theme`.
- THE SYSTEM SHALL apply the stored theme class to the `<html>` element before first paint to prevent flash of unstyled content.
- THE SYSTEM SHALL default to dark theme when no preference is stored.

### 1.2 Sidebar (Desktop)

The sidebar is visible at `md` breakpoint and above, fixed at `224px` (14rem / `w-56`) width.

- THE SYSTEM SHALL display the application brand label "Claude Code Lens" at the top of the sidebar.
- THE SYSTEM SHALL render the following navigation links in order:
  1. overview → `/`
  2. projects → `/projects`
  3. sessions → `/sessions`
  4. costs → `/costs`
  5. tools → `/tools`
  6. activity → `/activity`
  7. history → `/history`
  8. todos → `/todos`
  9. plans → `/plans`
  10. memory → `/memory`
  11. settings → `/settings`
  12. export → `/export`
- THE SYSTEM SHALL visually distinguish the active navigation link from inactive links.
- THE SYSTEM SHALL display an attribution label "Made by Arindam" at the bottom of the sidebar.
- THE SYSTEM SHALL display a theme-toggle button in the sidebar.

### 1.3 Bottom Navigation (Mobile)

The bottom nav is visible below the `md` breakpoint.

- THE SYSTEM SHALL display the following links in the bottom nav: home (`/`), sessions, costs, projects, activity.
- THE SYSTEM SHALL display a theme-toggle button in the bottom nav.

### 1.4 TopBar

Every page renders a `TopBar` component as a sticky page header.

- THE SYSTEM SHALL display a `title` and `subtitle` in the TopBar for each page (values specified per-page below).
- THE SYSTEM SHALL display a "last update" timestamp showing the current time, updated in real time.
- THE SYSTEM SHALL display a global search trigger button labelled "Search" (keyboard shortcut ⌘K / Ctrl+K).
- THE SYSTEM SHALL display a "refresh charts" button that re-fetches all data on the current page.
- WHERE the page is the Overview, THE SYSTEM SHALL display a "Star on GitHub" link in the TopBar pointing to the repository URL.

### 1.5 Global Search (⌘K)

- THE SYSTEM SHALL provide a global command-palette search accessible via ⌘K (macOS) / Ctrl+K (other platforms) or by clicking the "Search" button.
- THE SYSTEM SHALL allow the user to search across sessions and navigate directly to a session detail page from search results.
- THE SYSTEM SHALL close the search palette when the user presses Escape.

### 1.6 Theme Toggle

- THE SYSTEM SHALL toggle between `dark` and `light` themes when the theme-toggle button is activated.
- THE SYSTEM SHALL persist the selected theme across page reloads via `localStorage`.

### 1.7 Error and Loading States

- WHILE data is loading from the API, THE SYSTEM SHALL display skeleton placeholder elements (animated pulse) in place of content.
- WHEN an API request fails, THE SYSTEM SHALL display a red error message with the error description in monospace font.

---

## 2. Overview Page (`/`)

**TopBar**: title `"claude-code-analytics"`, subtitle `"real-time monitoring dashboard"`, show GitHub star button.

**Data sources**: `GET /api/stats`, `GET /api/sessions`, `GET /api/projects` (all polled every 5 seconds).

### 2.1 Summary Statistics Row

- THE SYSTEM SHALL display the following stats in a horizontal row:
  - **conversations**: total message count (formatted with locale separators)
  - **claude sessions**: total session count; sub-label showing `"this month: N · this week: N"`
  - **tokens**: total token count (input + output + cache_read + cache_write), formatted with SI suffix (e.g. `17.4B`)
  - **projects**: total project count
  - **storage**: `~/.claude/` directory size in human-readable bytes

### 2.2 Date Range Filter

- THE SYSTEM SHALL display two text inputs labelled `"from:"` and `"to:"` with placeholder `"MM/DD/YYYY"`.
- THE SYSTEM SHALL default `from` to 7 days before today and `to` to today.
- THE SYSTEM SHALL use the date range to compute the number of days shown in the Token Usage Over Time chart (`chartDays = max(1, min(365, diff))`).
- IF either date input cannot be parsed, THE SYSTEM SHALL fall back to 90 days for the chart window.

### 2.3 Token Usage Over Time Chart

- THE SYSTEM SHALL render a bar chart titled "Token usage over time" showing daily message count and session count over the selected date window.
- THE SYSTEM SHALL display a legend with entries: `messages`, `sessions`.
- THE SYSTEM SHALL label the X-axis with formatted dates (e.g. "Apr 4").

### 2.4 Project Activity Distribution Chart

- THE SYSTEM SHALL render a donut chart titled "Project activity distribution" showing the share of sessions by project.
- THE SYSTEM SHALL aggregate projects beyond the top N into an "others" segment.
- THE SYSTEM SHALL display a colour-coded legend listing project names.

### 2.5 Peak Hours Chart

- THE SYSTEM SHALL render a bar chart titled "Peak hours" covering all 24 hours (0–23).
- THE SYSTEM SHALL visually highlight the top 3 peak hours.
- THE SYSTEM SHALL display the label "top 3 peak hours highlighted" below the chart.
- THE SYSTEM SHALL label hour-axis ticks in 12-hour format (e.g. "10a", "11p").

### 2.6 Model Distribution Chart

- THE SYSTEM SHALL render a donut chart titled "Model distribution" showing session share per Claude model string.
- THE SYSTEM SHALL display a colour-coded legend listing model names.

### 2.7 Token Breakdown Bar

- THE SYSTEM SHALL render a segmented horizontal bar showing proportional widths for: `input`, `output`, `cache_read`, `cache_write` tokens.
- THE SYSTEM SHALL display each segment's label, formatted token count, and percentage below the bar.
- Token colours: input `#60a5fa`, output `#d97706`, cache_read `#34d399`, cache_write `#a78bfa`.

### 2.8 Recent Conversations Table

- THE SYSTEM SHALL render a table titled "Recent conversations" showing the most recent sessions.
- THE SYSTEM SHALL provide four filter buttons: **active**, **recent**, **inactive**, **all**.
- THE SYSTEM SHALL display the following columns:
  | Column | Description |
  |---|---|
  | conversation id | Truncated UUID (first 8 chars), link to `/sessions/[id]` |
  | project | Project display name, link to `/sessions/[id]` |
  | model | Truncated model name |
  | messages | Total message count |
  | tokens | Total tokens (formatted) |
  | last activity | Relative timestamp (e.g. "40m ago") |
  | conversation state | State label (e.g. "Completed") |
  | status | Activity status badge (active / inactive) |

---

## 3. Projects Page (`/projects`)

**TopBar**: title `"claude-code-analytics · projects"`, subtitle `"{n} projects"` (or `"loading..."` while fetching).

**Data source**: `GET /api/projects` (polled every 5 seconds).

### 3.1 Filters and Sorting

- THE SYSTEM SHALL display a search text input with placeholder `"Search projects..."`.
- THE SYSTEM SHALL filter projects by `display_name` and `project_path` (case-insensitive substring match).
- THE SYSTEM SHALL display four sort buttons: **Recent** (default), **Cost**, **Sessions**, **Time**.
  - Recent: sort by `last_active` descending
  - Cost: sort by `estimated_cost` descending
  - Sessions: sort by `session_count` descending
  - Time: sort by `total_duration_minutes` descending
- THE SYSTEM SHALL visually indicate the active sort button.

### 3.2 Project Grid

- THE SYSTEM SHALL display projects as a responsive card grid: 1 column (default), 2 columns (`lg`), 3 columns (`xl`).
- WHILE loading, THE SYSTEM SHALL display 6 skeleton card placeholders.

### 3.3 Project Card

Each card is a link to `/projects/[slug]` and displays:

- **Display name** (heading)
- **Relative last-active timestamp** (e.g. "1h ago", "2 days ago")
- **Full project path**
- **Feature badges** (shown only when true):
  - `🔌 mcp` when `uses_mcp`
  - `🤖 agent` when `uses_task_agent`
- **Session count**: `"{n} sessions"`
- **Message count**: `"{n} msgs"` (formatted with locale separators for large values)
- **Duration**: formatted as `"Xh Ym"` or `"<1m"`
- **Top tools**: tool name + call count, shown for the most-used tools (up to 5)
- **Branch list**: up to 10 git branch names
- **Estimated cost**: `"est. cost"` label + `"$X.XX"` value

### 3.4 Empty State

- WHEN no projects match the search, THE SYSTEM SHALL display `"No projects match your search."`.
- WHEN no projects are found, THE SYSTEM SHALL display `"No projects found in ~/.claude/"`.

---

## 4. Sessions Page (`/sessions`)

**TopBar**: title `"claude-code-analytics · sessions"`, subtitle `"{n} total sessions"`.

**Data source**: `GET /api/sessions` (polled every 5 seconds).

### 4.1 Filters

- THE SYSTEM SHALL display a search text input with placeholder `"Search project or prompt..."` that filters by project path and first prompt.
- THE SYSTEM SHALL display three toggle checkboxes (default on):
  - `⚡ compacted` — show/hide sessions with context compaction
  - `🤖 agent` — show/hide sessions using task agents
  - `🔌 mcp` — show/hide sessions using MCP
- THE SYSTEM SHALL display the current filtered session count: `"{n} sessions"`.

### 4.2 Sessions Table

- THE SYSTEM SHALL display sessions in a table with the following columns:
  | Column | Sortable | Description |
  |---|---|---|
  | Date | Yes (↓ default) | Session start date |
  | Project | No | Full project path, link to `/sessions/[id]` |
  | (prompt) | No | First prompt excerpt |
  | Dur | Yes | Duration formatted (`Xh Ym` or `<1m`) |
  | Msgs | Yes | Total message count |
  | Tools | No | Total tool call count |
  | Cost | Yes | Estimated cost (`$X.XX`) |
  | Flags | No | Feature badge icons |
- THE SYSTEM SHALL indicate the active sort column with a directional arrow.
- THE SYSTEM SHALL display the following flag badges where applicable:
  - `⚡ compacted`
  - `🤖 agent`
  - `🔌 mcp`
  - `🔍 web` (uses web search or web fetch)
  - `🧠 thinking` (has thinking turns)

### 4.3 Pagination

- THE SYSTEM SHALL paginate the sessions list with pages of 25 rows.
- THE SYSTEM SHALL display a pagination bar showing: `"Page X of Y · N sessions"`, previous (←) and next (→) buttons, and up to 5 direct page number buttons.
- WHEN on the first page, THE SYSTEM SHALL disable the previous button.
- WHEN on the last page, THE SYSTEM SHALL disable the next button.

---

## 5. Session Detail Page (`/sessions/[id]`)

**TopBar**: title `"session replay"`, subtitle = session ID.

**Data sources**: `GET /api/sessions/[id]/replay` (single load), `GET /api/sessions/[id]` (single load).

### 5.1 Stats Bar

Below the TopBar, THE SYSTEM SHALL display a stats bar containing:
- **turns**: count of assistant-type turns
- **tokens**: total tokens (input + output + cache), formatted
- **cost**: total estimated cost (`$X.XX`)
- **duration**: formatted from `duration_minutes`
- **compactions**: count of compaction events (shown only if > 0), with `⚡` prefix

### 5.2 Two-Column Layout

- THE SYSTEM SHALL render the session detail in a two-column layout:
  - **Left (main)**: scrollable conversation replay
  - **Right (sidebar)**: session metadata and charts

### 5.3 Conversation Replay

- THE SYSTEM SHALL render each turn in the `turns` array as a visually distinct card:
  - **User turn card**: displays user message text
  - **Assistant turn card**: displays assistant text, tool calls, tool results, and optional thinking blocks
- THE SYSTEM SHALL render thinking blocks as collapsible sections with toggle button labelled `"🧠 thinking block ▾"`, collapsed by default.
- THE SYSTEM SHALL render tool calls with tool name and input parameters.
- THE SYSTEM SHALL render tool results with result content and error styling for failed calls.
- THE SYSTEM SHALL render compaction events inline in the conversation at their correct chronological position with `⚡` indicator and optional summary text.

### 5.4 Session Sidebar

- THE SYSTEM SHALL display session badges (feature flags) in the sidebar.
- THE SYSTEM SHALL display a token accumulation chart (cumulative tokens over turns).
- THE SYSTEM SHALL display session metadata: start time, project, model, git branch, Claude Code version.

---

## 6. Project Detail Page (`/projects/[slug]`)

**TopBar**: title = project display name, subtitle = project path.

**Data source**: `GET /api/projects/[slug]` (single load).

### 6.1 Summary Stats

- THE SYSTEM SHALL display summary stat cards for: sessions, total messages, total duration, estimated cost, input tokens, output tokens, git commits, git pushes, files modified, lines added, lines removed.

### 6.2 Languages and Tools

- THE SYSTEM SHALL display a breakdown of programming languages detected in the project.
- THE SYSTEM SHALL display a tool usage breakdown with tool name and call count.

### 6.3 Sessions List

- THE SYSTEM SHALL display a filterable, sortable list of sessions belonging to this project, consistent with the Sessions page table format.

### 6.4 Branch List

- THE SYSTEM SHALL display up to 10 git branches associated with the project.

---

## 7. Costs Page (`/costs`)

**TopBar**: title `"claude-code-analytics · costs"`, subtitle `"estimated spend from ~/.claude/"`.

**Data source**: `GET /api/costs` (polled every 5 seconds).

### 7.1 Summary Stats

- THE SYSTEM SHALL display: total estimated cost and total cache savings.

### 7.2 Cost Over Time Chart

- THE SYSTEM SHALL render a stacked bar chart titled "Cost Over Time" showing daily cost split by model.
- THE SYSTEM SHALL label the X-axis with dates and the Y-axis with dollar amounts.

### 7.3 Cost by Project Chart

- THE SYSTEM SHALL render a bar chart titled "Cost by Project" showing estimated cost per project, sorted descending.

### 7.4 Per-Model Token Table

- THE SYSTEM SHALL render a table titled "Per-Model Token Breakdown" with columns:
  - Model name, input tokens, output tokens, cache write tokens, cache read tokens, estimated cost, cache savings, cache hit rate (%)

### 7.5 Cache Efficiency Panel

- THE SYSTEM SHALL render a "Cache Efficiency" panel showing overall cache hit rate, total savings in dollars, and per-model cache efficiency metrics.

### 7.6 Error State

- WHEN the API returns a 404 (stats-cache.json not found), THE SYSTEM SHALL display an appropriate error message.

---

## 8. Activity Page (`/activity`)

**TopBar**: title `"claude-code-analytics · activity"`.

**Data source**: `GET /api/activity` (polled every 5 seconds).

### 8.1 Activity Heatmap

- THE SYSTEM SHALL render a GitHub-style contribution heatmap showing daily session/message activity over the past 52 weeks (approximately 1 year).
- THE SYSTEM SHALL colour cells by relative activity level using a green gradient (no activity = muted, high activity = bright green).

### 8.2 Streak Card

- THE SYSTEM SHALL display a streak card showing:
  - **Current streak**: consecutive active days up to today
  - **Longest streak**: longest historical consecutive active-day run

### 8.3 Summary Stats

- THE SYSTEM SHALL display: total active days, most active day (date), message count on most active day.

### 8.4 Peak Hours Chart

- THE SYSTEM SHALL render the same 24-hour peak hours chart as on the Overview page.

### 8.5 Day-of-Week Chart

- THE SYSTEM SHALL render a bar chart showing activity by day of week (Sun–Sat) using message or session counts.

### 8.6 Usage Over Time Chart

- THE SYSTEM SHALL render the same token/session usage over time chart as on the Overview page, without the date range filter.

---

## 9. Tools Page (`/tools`)

**TopBar**: title `"claude-code-analytics · tools & features"`, subtitle `"every tool call, MCP server, and feature"`.

**Data source**: `GET /api/tools` (polled every 5 seconds).

### 9.1 Summary Stat

- THE SYSTEM SHALL display total tool call count.

### 9.2 Tool Ranking Chart

- THE SYSTEM SHALL render a horizontal bar chart of all tools ranked by `total_calls`, coloured by tool category.
- THE SYSTEM SHALL display a category legend using `CATEGORY_LABELS` and `CATEGORY_COLORS`.

### 9.3 MCP Server Panel

- THE SYSTEM SHALL render a panel listing each MCP server with: server name, total calls, session count, and a list of individual MCP tools with call counts.

### 9.4 Feature Adoption Table

- THE SYSTEM SHALL render a table titled "Feature Adoption" with rows for each feature:
  - task_agents, mcp, web_search, web_fetch, plan_mode, git_commits
  - Columns: feature name, sessions using feature (count), % of total sessions

### 9.5 Version History Table

- THE SYSTEM SHALL render a table of Claude Code versions seen across sessions with columns: version string, session count, first seen date, last seen date.

### 9.6 Branch Activity Table

- THE SYSTEM SHALL render a table of git branches with columns: branch name, turn count.

---

## 10. History Page (`/history`)

**TopBar**: title `"claude-code-lens · history"`, subtitle `"~/.claude/history.jsonl"`.

**Data source**: `GET /api/history?limit=2000` (polled every 30 seconds).

### 10.1 Search

- THE SYSTEM SHALL display a search text input that filters history entries by `display` text and `project` (case-insensitive substring).
- THE SYSTEM SHALL reset pagination to page 1 when the search changes.

### 10.2 History List

- THE SYSTEM SHALL display history entries in reverse chronological order (most recent first).
- THE SYSTEM SHALL paginate at 50 entries per page.
- Each entry card SHALL display:
  - `display` text (the command or prompt text)
  - Formatted timestamp (e.g. "Apr 9, 2026, 4:22 PM")
  - Project name (last path segment of `project`)

### 10.3 Pagination

- THE SYSTEM SHALL display a pagination bar with: previous button, `"{page} / {totalPages}"` label, next button.
- THE SYSTEM SHALL disable the previous button on page 1 and the next button on the last page.

### 10.4 Empty States

- WHEN no history entries are found, THE SYSTEM SHALL display `"No history found in ~/.claude/history.jsonl"`.
- WHEN no entries match the search, THE SYSTEM SHALL display `"No entries match your search."`.

---

## 11. Todos Page (`/todos`)

**TopBar**: title derived from data (source file name or "todos").

**Data source**: `GET /api/todos` (single load with manual refresh).

### 11.1 Filter Tabs

- THE SYSTEM SHALL display four filter tabs with item counts:
  - **all**, **pending**, **in_progress**, **completed**
- THE SYSTEM SHALL visually highlight the active filter tab.

### 11.2 Search

- THE SYSTEM SHALL display a search input with placeholder `"search todos..."` that filters by todo `content`.
- WHEN a filter or search is active, THE SYSTEM SHALL display `"showing {n} of {total} todos"`.

### 11.3 Todo Item Card

Each todo card SHALL display:
- **Status icon** in a coloured circle (pending: hollow, in_progress: spinner/indicator, completed: checkmark)
- **Content text** (strikethrough and muted when `status === "completed"`)
- **Status badge** (coloured pill label)
- **Priority badge** if `priority` is set: high (red), medium (yellow), low (muted)
- **Source file name** (the `.json` filename the todo was read from)
- **Date** (formatted modification time of the source file)

### 11.4 Empty State

- WHEN the filtered list is empty, THE SYSTEM SHALL display a `"✓"` icon and an appropriate message.

---

## 12. Plans Page (`/plans`)

**TopBar**: title `"claude-code-lens · plans"`, subtitle `"~/.claude/plans/"`.

**Data source**: `GET /api/plans` (single load with manual refresh).

### 12.1 Search and Count

- THE SYSTEM SHALL display a search input with placeholder `"search plans by name or content..."`.
- THE SYSTEM SHALL display `"{n} plans"` (or `"{filtered} of {total} plans"` when filtered).

### 12.2 Plan Card

Each plan card SHALL:
- Display a header with: `📋` icon, plan title (extracted from first `#` heading, or filename as fallback), word count, modification date.
- Be expandable/collapsible via a click on the header.
- Show a preview of the first 12 non-empty lines when collapsed.
- Show a "more" indicator when there are more than 12 lines.
- Render the full plan content as Markdown when expanded, using an inline renderer that supports: headings (h1–h2), bold (`**`), inline code, fenced code blocks, ordered lists, unordered lists, horizontal rules.

---

## 13. Memory Page (`/memory`)

**TopBar**: title derived from application.

**Data source**: `GET /api/memory` (polled every 15 seconds), `PATCH /api/memory` (for edits).

### 13.1 Summary Stats Row

- THE SYSTEM SHALL display stat cards for: total memories, user count, feedback count, project count, stale (>30 days old) count.

### 13.2 Type Filter Tabs

- THE SYSTEM SHALL display filter tabs for: **all**, **user**, **feedback**, **project**, **reference**, **index** (each with count).
- Memory type colours: user=blue (`#60a5fa`), feedback=red (`#f87171`), project=green (`#34d399`), reference=teal, index=yellow (`#fbbf24`), unknown=gray.

### 13.3 Search

- THE SYSTEM SHALL display a search input with placeholder `"search memories..."` that filters by file name, content, and project slug.

### 13.4 Memory Entry Card

Each memory card SHALL display:
- **Type badge** (coloured, uppercase label)
- **File name**
- **Project slug** (abbreviated) + short path
- **Relative modification date** (e.g. "3 days ago")
- **Expandable content**: clicking the card reveals the full markdown content of the memory file.

### 13.5 Stale Indicator

- THE SYSTEM SHALL visually distinguish memory entries whose `mtime` is more than 30 days ago.

---

## 14. Settings Page (`/settings`)

**TopBar**: title `"claude-code-lens · settings"`, subtitle `"~/.claude/settings.json"`.

**Data source**: `GET /api/settings` (single load with manual refresh).

### 14.1 Storage Section

- THE SYSTEM SHALL display a "Storage" section with the total bytes used by `~/.claude/` in human-readable format and the label `"used by ~/.claude/"`.

### 14.2 Settings JSON Viewer

- THE SYSTEM SHALL display the raw `settings.json` content as a structured tree view with labelled sections including: `env`, `permissions` (allow/deny arrays), `model`, `hooks`, `statusLine`, `enabledPlugins`, `sandbox`, and any other present keys.
- THE SYSTEM SHALL display string values in quotes.
- The settings viewer is read-only (no editing).

### 14.3 Environment Variables Section

- THE SYSTEM SHALL display a dedicated "Environment Variables" section listing each key-value pair from `settings.json.env`.

### 14.4 Skills Section

- THE SYSTEM SHALL display a "Skills ({n})" section listing all installed skills with: skill name, skill description/title.

### 14.5 Plugins Section

- THE SYSTEM SHALL display a "Plugins ({n})" section listing all installed plugins with: plugin identifier, scope (`project` / `user`), version string, installation date.

---

## 15. Export / Import Page (`/export`)

**TopBar**: title derived from application.

### 15.1 Export Panel

- THE SYSTEM SHALL display an "Export" panel with:
  - Optional "From date" input (type=date)
  - Optional "To date" input (type=date)
  - A "⬇ Download Export (.ccboard.json)" button
- WHEN the export button is clicked, THE SYSTEM SHALL POST to `/api/export` with optional `dateRange` body, then trigger a browser file download of the JSON response as `*.ccboard.json`.
- WHILE exporting, THE SYSTEM SHALL disable the button and show `"⏳ Exporting..."`.

### 15.2 Import Panel

- THE SYSTEM SHALL display an "Import / Merge from Another Machine" panel.
- THE SYSTEM SHALL provide a drag-and-drop zone that accepts `.ccboard.json` files; the zone SHALL visually highlight on dragover.
- THE SYSTEM SHALL also accept file selection via a hidden file input triggered by clicking the zone.
- WHEN a file is selected or dropped, THE SYSTEM SHALL POST it to `/api/import` and display a preview diff showing:
  - total_in_export, already_present, new_sessions counts
  - A list of new sessions to be added
- THE SYSTEM SHALL provide a "Confirm Import" button to proceed.
- THE SYSTEM SHALL display an error message if the import fails or the file is malformed.
- The import is **additive-only** — existing sessions are never overwritten. THE SYSTEM SHALL communicate this clearly to the user in the UI copy.
