# CC-Lens Dashboard — API Contract

> **Scope**: Contract between the frontend dashboard and the backend API. This document defines all endpoints, request shapes, response shapes, and error conditions.
> **Base URL**: `/api` (relative to the application root, e.g. `http://localhost:3000/api`)
> **Protocol**: HTTP/1.1, JSON body (`Content-Type: application/json`). All endpoints are `force-dynamic` (no caching at the Next.js layer).
> **Polling intervals**: Listed per-endpoint; the UI uses SWR with these intervals.

---

## Common Conventions

- All responses return `Content-Type: application/json`.
- All error responses return `{ "error": "<message>" }` with an appropriate HTTP status code.
- Timestamps are ISO 8601 strings (e.g. `"2026-04-09T14:22:00.000Z"`) unless otherwise noted.
- Dates (no time) are `"YYYY-MM-DD"` strings.
- Token counts are plain integers.
- Costs are floating-point USD amounts (e.g. `25.66`).
- Durations are floating-point minutes.

---

## Endpoints

### `GET /api/stats`

**Used by**: Overview page  
**Poll interval**: 5 seconds

#### Response `200 OK`

```ts
{
  stats: {
    version: number
    lastComputedDate: string           // "YYYY-MM-DD"
    dailyActivity: Array<{
      date: string                     // "YYYY-MM-DD"
      messageCount: number
      sessionCount: number
      toolCallCount: number
    }>
    tokensByDate: Array<{
      date: string
      [modelKey: string]: number | string  // model usage by date
    }>
    dailyModelTokens?: Array<{
      date: string
      [modelKey: string]: number | string
    }>
    modelUsage: Record<string, {       // keyed by model string
      inputTokens: number
      outputTokens: number
      cacheCreationInputTokens: number
      cacheReadInputTokens: number
    }>
    totalSessions: number
    totalMessages: number
    longestSession: {
      sessionId: string
      duration: number                 // minutes
      messageCount: number
      timestamp: string
    }
    firstSessionDate: string
    hourCounts: Record<string, number> // "0".."23" → count
    totalSpeculationTimeSavedMs: number
  }
  computed: {
    totalCost: number
    totalCacheSavings: number
    totalTokens: number
    totalInputTokens: number
    totalOutputTokens: number
    totalCacheReadTokens: number
    totalCacheWriteTokens: number
    totalToolCalls: number
    activeDays: number
    avgSessionMinutes: number
    sessionsThisMonth: number
    sessionsThisWeek: number
    storageBytes: number
    sessionCount: number
  }
}
```

**Notes**:
- `dailyActivity` is computed from session JSONL files directly, merged with `stats-cache.json`. Session-derived data overrides stats-cache data for the same date.
- `modelUsage` keys are raw model identifier strings as reported in JSONL (e.g. `"claude-sonnet-4-6"`, `"text-max"`).
- IF `stats-cache.json` is not found, THE SERVER SHALL return a synthetic stats object derived from session JSONL only; the `stats` object will have `version: 0` and empty fields for data that requires stats-cache.

---

### `GET /api/sessions`

**Used by**: Overview page, Sessions page  
**Poll interval**: 5 seconds

#### Response `200 OK`

```ts
{
  sessions: Array<SessionWithFacet>
}
```

Where `SessionWithFacet` extends `SessionMeta` with optional facet fields:

```ts
interface SessionMeta {
  session_id: string
  project_path: string
  start_time: string                   // ISO 8601
  duration_minutes: number
  user_message_count: number
  assistant_message_count: number
  tool_counts: Record<string, number>  // tool_name → count
  languages: Record<string, number>    // language → file count
  git_commits: number
  git_pushes: number
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens?: number
  cache_read_input_tokens?: number
  first_prompt: string
  user_interruptions: number
  user_response_times: number[]
  tool_errors: number
  tool_error_categories: Record<string, number>
  uses_task_agent: boolean
  uses_mcp: boolean
  uses_web_search: boolean
  uses_web_fetch: boolean
  lines_added: number
  lines_removed: number
  files_modified: number
  message_hours: number[]
  user_message_timestamps: string[]
}

// Facet fields (may be undefined if facet file not present)
interface Facet {
  session_id: string
  underlying_goal: string
  goal_categories: Record<string, number>
  outcome: string
  user_satisfaction_counts: Record<string, number>
  claude_helpfulness: string
  session_type: string
}
```

**Notes**:
- Sessions are returned sorted by `start_time` descending (most recent first).
- `tool_counts` uses the raw tool name strings as keys (e.g. `"Bash"`, `"mcp__linear_linear__get_issue"`).

---

### `GET /api/sessions/[id]`

**Used by**: Session detail page  
**Poll interval**: None (single load)

#### Path parameters
- `id`: session UUID string

#### Response `200 OK`

```ts
{
  session: SessionMeta & {
    estimated_cost: number
  }
}
```

#### Response `404 Not Found`

```json
{ "error": "Session not found" }
```

---

### `GET /api/sessions/[id]/replay`

**Used by**: Session detail page  
**Poll interval**: None (single load)

#### Path parameters
- `id`: session UUID string

#### Response `200 OK`

```ts
{
  session_id: string
  slug?: string                        // project slug
  version?: string                     // Claude Code version
  git_branch?: string
  turns: Array<ReplayTurn>
  compactions: Array<CompactionEvent>
  summaries: Array<SummaryEvent>
  total_cost: number
}

interface ReplayTurn {
  uuid: string
  parentUuid: string | null
  type: 'user' | 'assistant'
  timestamp: string
  model?: string
  usage?: {
    input_tokens: number
    output_tokens: number
    cache_creation_input_tokens?: number
    cache_read_input_tokens?: number
  }
  text?: string
  tool_calls?: Array<{
    id: string
    name: string
    input: Record<string, unknown>
  }>
  tool_results?: Array<{
    tool_use_id: string
    content: string
    is_error: boolean
  }>
  has_thinking?: boolean
  thinking_text?: string
  estimated_cost?: number
  turn_duration_ms?: number
  response_time_s?: number
}

interface CompactionEvent {
  uuid: string
  timestamp: string
  trigger: 'auto' | 'manual'
  pre_tokens: number
  summary?: string
  turn_index: number
}

interface SummaryEvent {
  uuid: string
  summary: string
  leaf_uuid: string
}
```

#### Response `404 Not Found`

```json
{ "error": "Session JSONL not found" }
```

---

### `GET /api/projects`

**Used by**: Overview page, Projects page  
**Poll interval**: 5 seconds

#### Response `200 OK`

```ts
{
  projects: Array<ProjectSummary>
}

interface ProjectSummary {
  slug: string                         // URL-safe path-derived identifier
  project_path: string                 // absolute filesystem path
  display_name: string                 // last segment of project_path
  session_count: number
  total_messages: number
  total_duration_minutes: number
  total_lines_added: number
  total_lines_removed: number
  total_files_modified: number
  git_commits: number
  git_pushes: number
  estimated_cost: number
  input_tokens: number
  output_tokens: number
  languages: Record<string, number>    // language → line/file count
  tool_counts: Record<string, number>
  last_active: string                  // ISO 8601
  first_active: string                 // ISO 8601
  uses_mcp: boolean
  uses_task_agent: boolean
  branches: string[]                   // up to 10 branch names
}
```

**Notes**:
- Sorted by `last_active` descending.
- `slug` is derived from the filesystem path (forward slashes replaced with `-`, leading `/` removed).

---

### `GET /api/projects/[slug]`

**Used by**: Project detail page  
**Poll interval**: None (single load)

#### Path parameters
- `slug`: URL-safe project identifier

#### Response `200 OK`

```ts
{
  project_path: string
  display_name: string
  sessions: Array<SessionMeta & {
    estimated_cost: number
    branch?: string
    version?: string
    has_compaction?: boolean
  }>
}
```

#### Response `404 Not Found`

```json
{ "error": "Project not found" }
```

---

### `GET /api/costs`

**Used by**: Costs page  
**Poll interval**: 5 seconds

#### Response `200 OK`

```ts
{
  total_cost: number
  total_savings: number
  models: Array<ModelCostBreakdown>
  daily: Array<DailyCost>
  by_project: Array<ProjectCost>
}

interface ModelCostBreakdown {
  model: string
  input_tokens: number
  output_tokens: number
  cache_write_tokens: number
  cache_read_tokens: number
  estimated_cost: number
  cache_savings: number
  cache_hit_rate: number               // 0.0 – 1.0
}

interface DailyCost {
  date: string                         // "YYYY-MM-DD"
  costs: Record<string, number>        // model → cost
  total: number
}

interface ProjectCost {
  slug: string
  display_name: string
  estimated_cost: number
  input_tokens: number
  output_tokens: number
}
```

#### Response `404 Not Found`

```json
{ "error": "stats-cache.json not found" }
```

**Notes**:
- `models` sorted by `estimated_cost` descending.
- `by_project` aggregated from session data, sorted by `estimated_cost` descending.
- `daily` uses model-level token data from `stats-cache.json`; missing days are omitted.

---

### `GET /api/activity`

**Used by**: Activity page  
**Poll interval**: 5 seconds

#### Response `200 OK`

```ts
{
  daily_activity: Array<{
    date: string                       // "YYYY-MM-DD"
    messageCount: number
    sessionCount: number
    toolCallCount: number
  }>
  hour_counts: Array<{
    hour: number                       // 0–23
    count: number
  }>                                   // all 24 hours present
  dow_counts: Array<{
    day: string                        // "Sun" | "Mon" | ... | "Sat"
    count: number
  }>                                   // all 7 days present
  streaks: {
    current: number                    // consecutive active days ending today
    longest: number                    // longest historical consecutive run
  }
  most_active_day: string              // "YYYY-MM-DD"
  most_active_day_msgs: number
  total_active_days: number
}
```

---

### `GET /api/tools`

**Used by**: Tools page  
**Poll interval**: 5 seconds

#### Response `200 OK`

```ts
{
  tools: Array<ToolSummary>
  mcp_servers: Array<McpServerSummary>
  feature_adoption: Record<string, {
    sessions: number
    pct: number                        // 0.0 – 1.0
  }>
  versions: Array<VersionRecord>
  branches: Array<{
    branch: string
    turns: number
  }>
  error_categories: Record<string, number>
  total_tool_calls: number
  total_errors: number
}

interface ToolSummary {
  name: string
  category: string                     // categorized by lib/tool-categories
  total_calls: number
  session_count: number
  error_count: number
}

interface McpServerSummary {
  server_name: string
  tools: Array<{
    name: string
    calls: number
  }>
  total_calls: number
  session_count: number
}

interface VersionRecord {
  version: string
  session_count: number
  first_seen: string                   // "YYYY-MM-DD"
  last_seen: string                    // "YYYY-MM-DD"
}
```

**Notes**:
- `feature_adoption` keys: `task_agents`, `mcp`, `web_search`, `web_fetch`, `plan_mode`, `git_commits`.
- MCP tool names follow the pattern `mcp__<server>__<tool>`.
- `category` values are defined in `lib/tool-categories.ts`.

---

### `GET /api/history`

**Used by**: History page  
**Poll interval**: 30 seconds

#### Query parameters
- `limit` (optional, integer): maximum entries to return. The UI requests `limit=2000`.

#### Response `200 OK`

```ts
{
  history: Array<{
    display: string                    // command/prompt text
    timestamp: number                  // Unix epoch milliseconds
    project: string                    // absolute project path
    sessionId?: string                 // UUID if associated with a session
  }>
}
```

**Notes**:
- Entries are returned in chronological order (oldest first). The UI reverses for display.
- Source: `~/.claude/history.jsonl`

---

### `GET /api/todos`

**Used by**: Todos page  
**Poll interval**: None (manual refresh)

#### Response `200 OK`

```ts
{
  files: Array<{
    name: string                       // filename (e.g. "todos.json")
    data: unknown                      // raw parsed JSON
    mtime: string                      // ISO 8601 modification time
  }>
}
```

**Notes**:
- Each file's `data` may be an array of todo objects, or `{ todos: [...] }`.
- Individual todo objects have the shape:
  ```ts
  {
    id?: string
    content?: string
    status?: 'pending' | 'in_progress' | 'completed'
    priority?: 'high' | 'medium' | 'low'
    [key: string]: unknown
  }
  ```
- Source: todo JSON files in `~/.claude/` or per-project directories.

---

### `GET /api/plans`

**Used by**: Plans page  
**Poll interval**: None (manual refresh)

#### Response `200 OK`

```ts
{
  plans: Array<{
    name: string                       // filename (without path)
    content: string                    // full markdown text
    mtime: string                      // ISO 8601 modification time
  }>
}
```

**Notes**:
- Sorted by `mtime` descending (most recently modified first).
- Source: `~/.claude/plans/` directory (markdown files).

---

### `GET /api/memory`

**Used by**: Memory page  
**Poll interval**: 15 seconds

#### Response `200 OK`

```ts
{
  memories: Array<MemoryEntry>
}

interface MemoryEntry {
  id: string                           // unique identifier (e.g. slug/file)
  file: string                         // filename (e.g. "user_role.md")
  projectSlug: string                  // project directory slug
  type: MemoryType                     // see below
  name: string                         // from frontmatter
  description: string                  // from frontmatter
  content: string                      // full file content
  mtime: string                        // ISO 8601
}

type MemoryType = 'user' | 'feedback' | 'project' | 'reference' | 'index' | 'unknown'
```

**Notes**:
- Source: `~/.claude/projects/[slug]/memory/*.md` files.
- `type` is extracted from YAML frontmatter field `type` in each `.md` file.
- Entries with `mtime` older than 30 days are considered "stale" by the UI.

---

### `PATCH /api/memory`

**Used by**: Memory page (content editing)

#### Request body

```ts
{
  projectSlug: string   // must not contain path separators
  file: string          // must end with ".md", must not contain path separators
  content: string       // full replacement content
}
```

#### Response `200 OK`

```json
{ "ok": true }
```

#### Error responses

| Status | Condition |
|---|---|
| `400` | Missing required fields, non-`.md` file extension, or invalid path |
| `403` | Resolved path escapes `~/.claude/projects/` |
| `500` | Filesystem write error |

**Security**: THE SERVER SHALL validate that `projectSlug` and `file` contain no path separators (`/`, `\`) before constructing the filesystem path. THE SERVER SHALL verify the resolved path starts with `~/.claude/projects/` before writing.

---

### `GET /api/settings`

**Used by**: Settings page  
**Poll interval**: None (single load with manual refresh)

#### Response `200 OK`

```ts
{
  settings: Record<string, unknown>    // raw parsed ~/.claude/settings.json
  storageBytes: number
  skills: Array<{
    name: string
    description: string
  }>
  plugins: Array<{
    id: string                         // e.g. "typescript-lsp@claude-plugins-official"
    scope: 'user' | 'project'
    version: string
    installedAt: string               // ISO 8601 or date string
  }>
}
```

---

### `POST /api/export`

**Used by**: Export page

#### Request body

```ts
{
  dateRange?: {
    from?: string    // "YYYY-MM-DD"
    to?: string      // "YYYY-MM-DD"
  }
}
```

#### Response `200 OK`

```ts
{
  exportedAt: string        // ISO 8601
  version: string           // format version identifier
  stats: StatsCache | null
  sessions: SessionMeta[]
  facets: Facet[]
  history: HistoryEntry[]
}
```

**Notes**:
- IF `dateRange` is provided, THE SERVER SHALL filter `sessions` and `history` to the specified range.
- The response is meant to be downloaded as a `.ccboard.json` file.

---

### `POST /api/import`

**Used by**: Export/Import page

#### Request body

`multipart/form-data` with a file field containing a `.ccboard.json` file, **OR** `application/json` body matching the export payload schema.

#### Response `200 OK` (preview/dry-run)

```ts
{
  total_in_export: number
  already_present: number
  new_sessions: number
  sessions_to_add: SessionMeta[]
}
```

#### Response `200 OK` (confirmed import)

```ts
{
  imported: number
  skipped: number
}
```

**Notes**:
- THE SERVER SHALL implement additive-only merge: sessions already present (matched by `session_id`) are never overwritten.
- THE SERVER SHALL return a diff preview before committing the import.
