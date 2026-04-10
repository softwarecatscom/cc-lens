# CC-Lens Backend API — Requirements

> **Scope**: Backend API implementation requirements. Frontend implementation is out of scope for this document.
> **Discovery method**: Source code analysis of all `app/api/*/route.ts` files, `lib/claude-reader.ts`, `lib/pricing.ts`, `types/claude.ts`, and correlation with the running application.

---

## 1. General Requirements

### 1.1 Runtime and Framework

- THE SYSTEM SHALL implement all API routes as Next.js App Router route handlers (`app/api/.../route.ts`).
- ALL API routes SHALL set `export const dynamic = 'force-dynamic'` to disable response caching.
- ALL API routes SHALL respond with `Content-Type: application/json`.
- ALL routes SHALL handle unexpected errors by returning `{ "error": "<message>" }` with HTTP 500.

### 1.2 Data Source Root

- THE SYSTEM SHALL resolve all Claude data from the directory `~/.claude/` on the host filesystem (where `~` is the home directory of the user running the server process).
- THE SYSTEM SHALL use `os.homedir()` to resolve the home directory at request time.
- THE SYSTEM SHALL NOT cache filesystem reads across requests (each request re-reads fresh data).

### 1.3 No Authentication

- THE SYSTEM SHALL NOT implement authentication or authorization. All endpoints are accessible to any client on the same host.

### 1.4 Pricing

- THE SYSTEM SHALL implement static per-model pricing tables in `lib/pricing.ts` covering all known Claude model variants.
- THE SYSTEM SHALL compute `estimated_cost` as `(inputTokens × inputPrice + outputTokens × outputPrice + cacheCreationInputTokens × cacheWritePrice + cacheReadInputTokens × cacheReadPrice)`.
- THE SYSTEM SHALL compute `cache_savings` as `cacheReadInputTokens × (inputPrice − cacheReadPrice)`.
- THE SYSTEM SHALL apply pricing by normalizing model strings to known pricing keys (e.g. `"claude-sonnet-4-6"` maps to Sonnet 4.6 pricing).
- WHEN a model string is not recognized, THE SYSTEM SHALL return zero cost rather than error.

---

## 2. Data Sources

### 2.1 `~/.claude/` Directory Structure

The backend SHALL read from the following locations:

| Path | Content | Used by |
|---|---|---|
| `~/.claude/stats-cache.json` | Pre-computed aggregate statistics | `/api/stats`, `/api/costs` |
| `~/.claude/projects/[slug]/*.jsonl` | Session JSONL conversation files | `/api/sessions`, `/api/sessions/[id]`, `/api/sessions/[id]/replay`, `/api/projects`, `/api/tools`, `/api/activity` |
| `~/.claude/history.jsonl` | CLI command history | `/api/history` |
| `~/.claude/settings.json` | Claude Code settings | `/api/settings` |
| `~/.claude/skills/` | Installed skill files | `/api/settings` |
| `~/.claude/plugins/` | Installed plugin metadata | `/api/settings` |
| `~/.claude/plans/` | Markdown plan files | `/api/plans` |
| `~/.claude/projects/[slug]/todos*.json` | Todo JSON files | `/api/todos` |
| `~/.claude/projects/[slug]/memory/*.md` | Memory markdown files | `/api/memory` |

### 2.2 Session JSONL Format

Each `.jsonl` file in `~/.claude/projects/[slug]/` represents one session and contains one JSON object per line. Relevant line types include:

- **Message lines**: `{ "type": "message", "role": "user"|"assistant", "content": [...], "uuid": "...", "parentUuid": "...", "timestamp": "...", "model": "...", "usage": {...} }`
- **Tool use lines**: assistant content blocks of type `"tool_use"` with `id`, `name`, `input` fields.
- **Tool result lines**: user content blocks of type `"tool_result"` with `tool_use_id`, `content`, `is_error` fields.
- **Thinking blocks**: assistant content blocks of type `"thinking"` with `thinking` text field.
- **Compaction lines**: `{ "type": "compaction", "timestamp": "...", "trigger": "auto"|"manual", "pre_tokens": N, "summary": "..." }`
- **Summary lines**: `{ "type": "summary", "summary": "...", "leafUuid": "..." }`
- **Metadata lines**: lines with `gitBranch`, `version`, `slug`, `sessionId` fields (may appear at start of file).

### 2.3 `stats-cache.json` Format

```ts
{
  version: number
  lastComputedDate: string
  dailyActivity: Array<{ date: string; messageCount: number; sessionCount: number; toolCallCount: number }>
  tokensByDate: Array<{ date: string; [model: string]: number | string }>
  dailyModelTokens?: Array<{ date: string; [model: string]: number | string }>
  modelUsage: Record<string, {
    inputTokens: number
    outputTokens: number
    cacheCreationInputTokens: number
    cacheReadInputTokens: number
  }>
  totalSessions: number
  totalMessages: number
  longestSession: { sessionId: string; duration: number; messageCount: number; timestamp: string }
  firstSessionDate: string
  hourCounts: Record<string, number>
  totalSpeculationTimeSavedMs: number
}
```

### 2.4 Project Slug Derivation

- THE SYSTEM SHALL derive a project slug from an absolute filesystem path by replacing `/` path separators with `-` and removing the leading separator.
- Example: `/Users/christo/ws/my-app` → `-Users-christo-ws-my-app`
- THE SYSTEM SHALL be able to reverse this mapping to reconstruct the original path from a slug (used in `GET /api/projects/[slug]`).

---

## 3. `/api/stats` Requirements

- THE SYSTEM SHALL read `stats-cache.json` and all session JSONL files.
- THE SYSTEM SHALL compute `dailyActivity` fresh from session JSONL on every request.
- THE SYSTEM SHALL merge session-derived `dailyActivity` with stats-cache `dailyActivity`; where the same date appears in both sources, the session-derived value SHALL take precedence.
- THE SYSTEM SHALL compute `modelUsage` totals (input, output, cache_creation, cache_read tokens) from `stats-cache.json`.
- THE SYSTEM SHALL compute `totalCost` and `totalCacheSavings` by iterating `modelUsage` entries through the pricing table.
- THE SYSTEM SHALL compute `sessionsThisMonth` as sessions whose `start_time` falls in the current calendar month.
- THE SYSTEM SHALL compute `sessionsThisWeek` as sessions whose `start_time` falls within the last 7 days (Monday–Sunday or rolling 7 days).
- THE SYSTEM SHALL compute `storageBytes` as the total size of all files under `~/.claude/`.
- THE SYSTEM SHALL compute `activeDays` as the count of distinct dates with at least one session.
- THE SYSTEM SHALL compute `avgSessionMinutes` as the mean of `duration_minutes` across all sessions.
- IF `stats-cache.json` is absent, THE SYSTEM SHALL return a synthetic response derived entirely from session JSONL with `version: 0` and empty fields for stats-cache-only data.

---

## 4. `/api/sessions` Requirements

- THE SYSTEM SHALL scan all `~/.claude/projects/[slug]/*.jsonl` files.
- FOR EACH `.jsonl` file, THE SYSTEM SHALL parse the file to extract `SessionMeta` fields.
- THE SYSTEM SHALL compute `SessionMeta` fields from JSONL content as follows:
  - `session_id`: from `sessionId` metadata line or filename (UUID).
  - `project_path`: resolved from the project slug's directory name.
  - `start_time`: timestamp of the earliest message line.
  - `duration_minutes`: difference between last and first message timestamps in minutes.
  - `user_message_count`: count of `role: "user"` message lines.
  - `assistant_message_count`: count of `role: "assistant"` message lines.
  - `tool_counts`: sum of tool_use content blocks per tool name.
  - `languages`: aggregated from any language metadata in the session.
  - `git_commits`: count of `Bash(git commit...)` or commit tool calls.
  - `git_pushes`: count of `Bash(git push...)` tool calls.
  - `input_tokens` / `output_tokens` / `cache_creation_input_tokens` / `cache_read_input_tokens`: summed from `usage` fields of all assistant message lines.
  - `first_prompt`: text content of the first user message.
  - `user_interruptions`: count of user messages sent while an assistant response was in progress.
  - `tool_errors`: count of `tool_result` blocks with `is_error: true`.
  - `tool_error_categories`: counts of tool names with errors.
  - `uses_task_agent`: true if any `Agent` or `Task` tool calls appear.
  - `uses_mcp`: true if any `mcp__*` tool calls appear.
  - `uses_web_search`: true if `WebSearch` tool calls appear.
  - `uses_web_fetch`: true if `WebFetch` tool calls appear.
  - `lines_added` / `lines_removed` / `files_modified`: aggregated from any git diff metadata in the session.
  - `message_hours`: hour-of-day (0–23) for each user message timestamp.
  - `user_message_timestamps`: ISO 8601 timestamps of all user messages.
- THE SYSTEM SHALL return sessions sorted by `start_time` descending.
- THE SYSTEM SHALL read JSONL files in parallel across project directories using `Promise.all`.

---

## 5. `/api/sessions/[id]` Requirements

- THE SYSTEM SHALL locate the JSONL file for the given session ID by scanning project directories.
- THE SYSTEM SHALL parse `SessionMeta` from the JSONL file (same logic as `/api/sessions`).
- THE SYSTEM SHALL compute `estimated_cost` from the session's token usage and the pricing table.
- IF no JSONL file is found for the given ID, THE SYSTEM SHALL return HTTP 404.

---

## 6. `/api/sessions/[id]/replay` Requirements

- THE SYSTEM SHALL locate and parse the JSONL file for the given session ID.
- THE SYSTEM SHALL construct a `ReplayData` object by processing each line:
  - Message lines (role=user/assistant) → `ReplayTurn` entries.
  - Tool use content blocks → appended to the parent `ReplayTurn.tool_calls`.
  - Tool result content blocks → appended to the parent `ReplayTurn.tool_results` (matched by `tool_use_id`).
  - Thinking content blocks → set `has_thinking: true`, populate `thinking_text`.
  - Compaction lines → `CompactionEvent` entries with `turn_index` set to the index of the preceding turn.
  - Summary lines → `SummaryEvent` entries.
  - Metadata lines → populate `slug`, `version`, `git_branch` on the `ReplayData`.
- THE SYSTEM SHALL compute `total_cost` by summing `estimated_cost` across all assistant turns.
- THE SYSTEM SHALL compute `estimated_cost` per turn from the turn's `usage` and model via the pricing table.
- THE SYSTEM SHALL populate `turn_duration_ms` and `response_time_s` where available from message metadata.
- IF no JSONL file is found, THE SYSTEM SHALL return HTTP 404.

---

## 7. `/api/projects` Requirements

- THE SYSTEM SHALL list all project directories under `~/.claude/projects/`.
- FOR EACH project directory, THE SYSTEM SHALL resolve the original filesystem path from the slug.
- THE SYSTEM SHALL group sessions by `project_path`.
- FOR EACH project group, THE SYSTEM SHALL compute aggregate `ProjectSummary` fields:
  - `session_count`, `total_messages`, `total_duration_minutes`, `total_lines_added`, `total_lines_removed`, `total_files_modified`, `git_commits`, `git_pushes`, `estimated_cost`, `input_tokens`, `output_tokens`.
  - `languages`: merged map of language counts across all sessions.
  - `tool_counts`: merged map of tool call counts across all sessions.
  - `last_active`: ISO 8601 timestamp of the most recent session's start.
  - `first_active`: ISO 8601 timestamp of the earliest session's start.
  - `uses_mcp`: true if any session has `uses_mcp: true`.
  - `uses_task_agent`: true if any session has `uses_task_agent: true`.
  - `branches`: up to 10 unique git branch names seen in session JSONL files.
- THE SYSTEM SHALL gather branch names by reading `gitBranch` fields from JSONL metadata lines (excluding `"HEAD"`).
- THE SYSTEM SHALL return projects sorted by `last_active` descending.
- THE SYSTEM SHALL process project directories in parallel using `Promise.all`.

---

## 8. `/api/projects/[slug]` Requirements

- THE SYSTEM SHALL resolve the project path from the slug.
- THE SYSTEM SHALL filter sessions to those matching `project_path`.
- IF no sessions match exactly, THE SYSTEM SHALL fall back to matching sessions where `project_path` ends with `/{lastSegment}` of the resolved path.
- FOR EACH session JSONL in the project, THE SYSTEM SHALL read branch and version metadata.
- THE SYSTEM SHALL return the filtered session list with `estimated_cost`, `branch`, `version`, and `has_compaction` fields appended.
- IF no project is found, THE SYSTEM SHALL return HTTP 404.

---

## 9. `/api/costs` Requirements

- THE SYSTEM SHALL require `stats-cache.json` to be present; return HTTP 404 if absent.
- THE SYSTEM SHALL compute per-model `ModelCostBreakdown` from `stats-cache.json.modelUsage`.
- THE SYSTEM SHALL compute `DailyCost` entries from `stats-cache.json.dailyModelTokens` (or `tokensByDate`), applying pricing to each model's daily token counts.
- THE SYSTEM SHALL aggregate `by_project` cost from session JSONL files, applying pricing per session.
- THE SYSTEM SHALL return `models` sorted by `estimated_cost` descending.
- THE SYSTEM SHALL return `by_project` sorted by `estimated_cost` descending.

---

## 10. `/api/activity` Requirements

- THE SYSTEM SHALL read `stats-cache.json` for `dailyActivity` and `hourCounts`.
- THE SYSTEM SHALL read all session JSONL metadata for `start_time` values.
- THE SYSTEM SHALL compute day-of-week counts by extracting `getDay()` from each session's `start_time`.
- THE SYSTEM SHALL compute active dates as the set of distinct `start_time.slice(0,10)` values.
- THE SYSTEM SHALL compute streaks:
  - **Longest streak**: longest consecutive-day run in the sorted active-date set.
  - **Current streak**: count backwards from today; stop when a day has no activity.
- THE SYSTEM SHALL identify the most active day as the date with the highest `messageCount` in `dailyActivity`.
- THE SYSTEM SHALL return `hour_counts` as an array of 24 objects `{ hour: 0..23, count: N }`.
- THE SYSTEM SHALL return `dow_counts` as an array of 7 objects `{ day: "Sun"|..|"Sat", count: N }`.

---

## 11. `/api/tools` Requirements

- THE SYSTEM SHALL aggregate tool usage across all session metadata (`tool_counts` maps).
- THE SYSTEM SHALL categorize each tool using a category mapping (`lib/tool-categories.ts`) and return the category name per tool.
- THE SYSTEM SHALL identify MCP tools by matching the naming pattern `mcp__<server>__<tool>`.
- THE SYSTEM SHALL aggregate MCP server summaries: for each server, list tools + call counts, total calls, and distinct session count.
- THE SYSTEM SHALL compute `feature_adoption` for: `task_agents`, `mcp`, `web_search`, `web_fetch`, `plan_mode`, `git_commits` — as session counts and percentages.
- THE SYSTEM SHALL read all JSONL files to extract `version` and `gitBranch` metadata lines for the version history and branch activity tables.
- THE SYSTEM SHALL aggregate `versions`: group sessions by Claude Code version string, compute `session_count`, `first_seen`, `last_seen`.
- THE SYSTEM SHALL aggregate `branches`: map branch name → total turn count across all sessions.
- THE SYSTEM SHALL aggregate `error_categories` from `tool_error_categories` maps across all sessions.
- THE SYSTEM SHALL process JSONL files in parallel using `Promise.all`.

---

## 12. `/api/history` Requirements

- THE SYSTEM SHALL read `~/.claude/history.jsonl` line by line.
- Each line is a JSON object with at least `display: string`, `timestamp: number`, `project: string`, and optionally `sessionId: string`.
- THE SYSTEM SHALL apply the `limit` query parameter (default: no limit) to cap the number of entries returned.
- THE SYSTEM SHALL return entries in chronological order (ascending by `timestamp`).
- IF `history.jsonl` does not exist, THE SYSTEM SHALL return `{ history: [] }` (not an error).

---

## 13. `/api/todos` Requirements

- THE SYSTEM SHALL scan for todo files in `~/.claude/projects/[slug]/` directories (files matching `todos*.json` or similar patterns).
- FOR EACH todo file, THE SYSTEM SHALL parse the JSON content and return it with the filename and modification time.
- THE SYSTEM SHALL return all files regardless of their internal structure; the UI handles normalization.
- IF no todo files are found, THE SYSTEM SHALL return `{ files: [] }`.

---

## 14. `/api/plans` Requirements

- THE SYSTEM SHALL read all `.md` files from `~/.claude/plans/`.
- FOR EACH file, THE SYSTEM SHALL return the filename, full text content, and `mtime` (ISO 8601).
- THE SYSTEM SHALL return files sorted by `mtime` descending.
- IF `~/.claude/plans/` does not exist, THE SYSTEM SHALL return `{ plans: [] }`.

---

## 15. `/api/memory` Requirements

### 15.1 `GET /api/memory`

- THE SYSTEM SHALL scan `~/.claude/projects/[slug]/memory/*.md` across all project slugs.
- FOR EACH `.md` file, THE SYSTEM SHALL parse the YAML frontmatter to extract `name`, `description`, `type`.
- Supported `type` values: `user`, `feedback`, `project`, `reference`, `index`. Unknown values SHALL be normalized to `"unknown"`.
- THE SYSTEM SHALL return `mtime` as ISO 8601.
- THE SYSTEM SHALL return the full file `content` (including frontmatter).
- THE SYSTEM SHALL derive `id` as `"{slug}/{filename}"`.
- THE SYSTEM SHALL process project directories in parallel.

### 15.2 `PATCH /api/memory`

- THE SYSTEM SHALL require `projectSlug`, `file`, and `content` in the request body; return HTTP 400 if any are missing.
- THE SYSTEM SHALL reject any `file` value that does not end with `.md`; return HTTP 400.
- THE SYSTEM SHALL reject `projectSlug` or `file` values containing path separators (`/` or `\`); return HTTP 400.
- THE SYSTEM SHALL construct the target path as `~/.claude/projects/{projectSlug}/memory/{file}`.
- THE SYSTEM SHALL verify the resolved absolute path starts with `~/.claude/projects/`; return HTTP 403 if it does not (path traversal prevention).
- THE SYSTEM SHALL create parent directories as needed (`mkdir -p`).
- THE SYSTEM SHALL write `content` as UTF-8 to the target path.

---

## 16. `/api/settings` Requirements

- THE SYSTEM SHALL read and parse `~/.claude/settings.json`.
- THE SYSTEM SHALL compute `storageBytes` as the total size of all files under `~/.claude/`.
- THE SYSTEM SHALL read skill files from `~/.claude/skills/` to produce the skills list.
- THE SYSTEM SHALL read installed plugin metadata from `~/.claude/plugins/` to produce the plugins list.
- IF `settings.json` does not exist, THE SYSTEM SHALL return `{ settings: null, storageBytes: N, skills: [], plugins: [] }`.

---

## 17. `/api/export` Requirements

- THE SYSTEM SHALL accept a `POST` request with optional `dateRange: { from?: string, to?: string }` body.
- THE SYSTEM SHALL read: `stats-cache.json`, all session JSONL metadata, facet files, and `history.jsonl`.
- IF `dateRange.from` is set, THE SYSTEM SHALL exclude sessions with `start_time` before that date.
- IF `dateRange.to` is set, THE SYSTEM SHALL exclude sessions with `start_time` after that date.
- THE SYSTEM SHALL apply the same date filter to `history` entries.
- THE SYSTEM SHALL return the export payload with a current `exportedAt` ISO 8601 timestamp and a `version` identifier string.
- THE SYSTEM SHALL NOT write any files to disk; the response is purely returned to the client for download.

---

## 18. `/api/import` Requirements

- THE SYSTEM SHALL accept a `POST` request with a `.ccboard.json` payload (either `multipart/form-data` or `application/json`).
- THE SYSTEM SHALL parse and validate the payload against the `ExportPayload` schema; return HTTP 400 for invalid payloads.
- THE SYSTEM SHALL compute an `ImportDiff`:
  - `total_in_export`: count of `sessions` in the payload.
  - `already_present`: count of sessions whose `session_id` already exists in `~/.claude/`.
  - `new_sessions`: count of sessions not yet present.
  - `sessions_to_add`: the new session metadata objects.
- THE SYSTEM SHALL implement additive-only merge: existing sessions (matched by `session_id`) SHALL NEVER be modified or deleted.
- WHEN confirmed, THE SYSTEM SHALL write new session metadata into the appropriate `~/.claude/projects/[slug]/` directory.
- THE SYSTEM SHALL return the final count of imported and skipped sessions.

---

## 19. Performance Requirements

- THE SYSTEM SHALL process session JSONL files in parallel (not sequentially) using concurrent async I/O.
- THE SYSTEM SHALL complete responses for the `/api/stats`, `/api/sessions`, `/api/projects` endpoints within a reasonable time for up to 500 session files.
- THE SYSTEM SHALL NOT hold file handles open between requests.
- THE SYSTEM SHALL NOT maintain server-side state; each request reads fresh from the filesystem.
