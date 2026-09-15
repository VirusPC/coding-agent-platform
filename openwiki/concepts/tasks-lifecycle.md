---
type: concept
title: Task Lifecycle & Status Model
description: The single tasks row that anchors every agent run — its statuses (pending/processing/completed/error/stopped), fields, append-only logs JSON, soft-delete semantics, and the API guards that gate status transitions.
tags: [tasks, lifecycle, status, state-machine, soft-delete, rate-limit, logs, jsonb]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-3401b854c53347910d2c84ce
    resource: repo://app/api/tasks/%5BtaskId%5D/continue/route.ts
  - id: openwiki-source-969b6ae89465d380116226fc
    resource: repo://app/api/tasks/%5BtaskId%5D/merge-pr/route.ts
  - id: openwiki-source-7ba88dffbf982709a6072164
    resource: repo://app/api/tasks/%5BtaskId%5D/route.ts
  - id: openwiki-source-c7baae88c7e2880cc027c016
    resource: repo://app/api/tasks/route.ts
  - id: openwiki-source-d8833a44f288fa20597092dd
    resource: repo://lib/constants.ts
  - id: openwiki-source-526feb2334febba699169a23
    resource: repo://lib/db/schema.ts
  - id: openwiki-source-b251c9f870bdc2494ff96ac6
    resource: repo://lib/db/settings.ts
  - id: openwiki-source-64381ce4d223926c6fcd9f15
    resource: repo://lib/sandbox/sandbox-registry.ts
  - id: openwiki-source-7ece006b3c5e6a3e70e9b390
    resource: repo://lib/utils/rate-limit.ts
  - id: openwiki-source-553b6b82f8719a4f1f3940bf
    resource: repo://lib/utils/task-logger.ts
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Task Lifecycle & Status Model

The `tasks` table ([`lib/db/schema.ts`](repo://lib/db/schema.ts#L76-L114)) is the central unit of work for the coding-agent-template. One row represents one agent run — a user's prompt against a repo — and is the durable source of truth for execution state, sandbox reachability, PR outcome, conversation history, and soft-delete visibility. This page is the single place that ties together the `tasks` row, the five lifecycle statuses, the append-only logs JSONB column, the `deletedAt` tombstone, and the API guards (`POST /api/tasks`, `GET /api/tasks[/:taskId]`, `PATCH /api/tasks/:taskId`, `DELETE /api/tasks`, `DELETE /api/tasks/:taskId`) that constrain which transitions are legal.

For the phased runtime walkthrough that moves a row from `pending` to `completed` or `error`, see [Runtime Flow: Request → Sandbox → Agent → Push](../architecture/runtime-flow.md). For the sandbox row pointers (`sandboxId`, `sandboxUrl`) and the `keepAlive` flag's effect on shutdown, see [Sandbox Lifecycle](./sandbox-lifecycle.md).

## The row

Every agent run lives in one `tasks` row. The schema is intentionally wide because the row is the only durable handle on a long-running piece of work — the HTTP request that creates it returns within milliseconds, but the row may be in `processing` for many minutes while the worker and sandbox run.

| Field | Type | Purpose |
| --- | --- | --- |
| `id` | `text` PK | 12-character `nanoid` (or client-provided) — see [`lib/utils/id.ts`](repo://lib/utils/id.ts#L3-L5) |
| `userId` | `text` FK → `users.id` (`onDelete: cascade`) | Ownership — every API query scopes by this |
| `prompt` | `text` NOT NULL | The original user request (also persisted as the first `taskMessages` row, `role='user'`) |
| `title` | `text` | AI-generated title via `lib/utils/title-generator.ts`, with `createFallbackTitle(prompt)` as the fallback |
| `repoUrl` | `text` | Target repository URL (may be `null` for non-repo tasks) |
| `selectedAgent` | `text` enum | One of `claude`/`codex`/`copilot`/`cursor`/`gemini`/`opencode` (default `claude`) |
| `selectedModel` | `text` | Optional per-task model override |
| `installDependencies` | `boolean` | Whether the sandbox should auto-install |
| `maxDuration` | `integer` | Minutes — request value or `getMaxSandboxDuration(userId)` |
| `keepAlive` | `boolean` | If true, leave the sandbox running after a successful push |
| `enableBrowser` | `boolean` | Whether to install Chromium + the `agent-browser` CLI |
| `status` | `text` enum | The lifecycle state — see [Status model](#status-model) |
| `progress` | `integer` | Coarse percent 0–100 — written by `logger.updateProgress` |
| `logs` | `jsonb<LogEntry[]>` | Append-only event stream — see [The logs column](#the-logs-column) |
| `error` | `text` | Free-form error string written by the error path; also `'Task was stopped by user'` on a stop |
| `branchName` | `text` | AI-generated or `agent/<timestamp>-<id>` fallback |
| `sandboxId` / `sandboxUrl` | `text` | The `Sandbox.get()` reconnect handle + the preview iframe URL |
| `agentSessionId` | `text` | Persisted by `processTask` so the continue endpoint can `--resume` |
| `prUrl` / `prNumber` / `prStatus` / `prMergeCommitSha` | mixed | PR outcome — `prStatus` enum: `open`/`closed`/`merged` |
| `mcpServerIds` | `jsonb<string[]>` | IDs of MCP connectors the agent was given |
| `createdAt` / `updatedAt` | `timestamp` | `defaultNow()` |
| `completedAt` | `timestamp NULL` | Only written on user-initiated stop or PR merge — see [When `completedAt` is written](#when-completedat-is-written) |
| `deletedAt` | `timestamp NULL` | Soft-delete tombstone — see [Soft delete vs. `status='stopped'`](#soft-delete-vs-statusstopped) |

The four sibling tables that hang off `tasks`:

- **`taskMessages`** — FK to `tasks.id` (`onDelete: cascade`). One row per conversation turn (`role='user'|'agent'`); see [`lib/db/schema.ts`](repo://lib/db/schema.ts#L360-L370). Streaming agents update a single row identified by `agentMessageId` in place.
- **`connectors`** — joined via `tasks.mcpServerIds` (the IDs are stored, not a real FK, because the user may later delete a connector).
- **`accounts` / `keys` / `settings`** — owned by the user, not by the task; the task row references the user, not these directly.

## Status model

`tasks.status` is a PostgreSQL enum (`pending | processing | completed | error | stopped`, [`lib/db/schema.ts`](repo://lib/db/schema.ts#L90-L94)). Every code path that mutates `status` is one of three mechanisms:

1. **API handlers** — `POST /api/tasks` sets `pending` on insert; `PATCH /api/tasks/:taskId` with `action: 'stop'` sets `stopped`; `POST /api/tasks/:taskId/continue` rewinds a completed/error/stopped task to `processing` and clears `completedAt`.
2. **`TaskLogger.updateStatus(...)`** — the only sanctioned writer from inside the worker; see [`lib/utils/task-logger.ts`](repo://lib/utils/task-logger.ts#L107-L131). It accepts `pending | processing | completed | error` (note: `stopped` is *not* in the worker's vocabulary — see [Soft delete vs. `status='stopped'`](#soft-delete-vs-statusstopped)).
3. **The `processTaskWithTimeout` timeout race** — when the `setTimeout` reject wins the `Promise.race` in [`app/api/tasks/route.ts#L252-L332`](repo://app/api/tasks/route.ts#L252-L332), the timeout handler writes `status='error'` with the timeout message.

### State transitions

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: a semicolon inside a label breaks rendering; rephrase the label. -->
```text
stateDiagram-v2
    [*] --> pending: POST /api/tasks inserts row (status=pending, progress=0, logs=[])
    pending --> processing: processTask start (logger.updateStatus processing)
    pending --> processing: POST continue rewinds an existing task
    processing --> completed: pushChangesToBranch succeeded (logger.updateStatus completed)
    processing --> error: agent reported failure OR catch handler (logger.updateStatus error)
    processing --> error: Promise.race timeout (timeout handler writes error)
    processing --> stopped: PATCH action=stop (user-initiated; killSandbox)
    completed --> processing: POST continue rewinds a completed task (clears completedAt)
    error --> processing: POST continue rewinds an errored task
    stopped --> processing: POST continue rewinds a stopped task
    completed --> stopped: PATCH action=stop rejected if not processing (only from processing)
    completed --> tombstoned: DELETE /api/tasks/:taskId (soft delete: deletedAt=now)
    error --> tombstoned: DELETE /api/tasks/:taskId
    stopped --> tombstoned: DELETE /api/tasks/:taskId
    completed --> [*]: DELETE /api/tasks?action=completed (hard delete by status)
    error --> [*]: DELETE /api/tasks?action=failed (hard delete by status)
    stopped --> [*]: DELETE /api/tasks?action=stopped (hard delete by status)
```

*Diagram: the five `status` values plus the `deletedAt` tombstone, with arrows showing every transition the codebase performs.*

Two things are worth highlighting because they are easy to misread:

- **`pending` is not reachable from any other state.** Nothing in the codebase writes `status='pending'` after the initial insert. If a row is `processing` and the worker rewinds, it always goes back to `processing` (not `pending`) via `continue`. The state is effectively "never started" only.
- **`stopped` is not written by the worker.** A `PATCH /api/tasks/:taskId` with `action: 'stop'` ([`app/api/tasks/[taskId]/route.ts#L62-L110`](repo://app/api/tasks/[taskId]/route.ts#L62-L110)) is the *only* writer of `status='stopped'`. The worker detects `status === 'stopped'` at its `isTaskStopped` checkpoints and returns early without mutating state.

### When `completedAt` is written

`completedAt` is written in exactly three places, and the absence of `completedAt` is meaningful:

| Where | When | What it means |
| --- | --- | --- |
| `PATCH /api/tasks/:taskId` with `action: 'stop'` ([`app/api/tasks/[taskId]/route.ts#L75-L84`](repo://app/api/tasks/[taskId]/route.ts#L75-L84)) | User-initiated stop | The task ended because a human asked it to |
| `POST /api/tasks/:taskId/merge-pr` ([`app/api/tasks/[taskId]/merge-pr/route.ts#L75-L85`](repo://app/api/tasks/[taskId]/merge-pr/route.ts#L75-L85)) | PR merged on GitHub | The change is now in `main` |
| `POST /api/tasks/:taskId/continue` ([`app/api/tasks/[taskId]/continue/route.ts#L77-L85`](repo://app/api/tasks/[taskId]/continue/route.ts#L77-L85)) | Follow-up message arrives | The row is being reanimated; `completedAt` is cleared back to `NULL` |

The worker's own success path (`logger.updateStatus('completed')` at [`app/api/tasks/route.ts#L692`](repo://app/api/tasks/route.ts#L692)) **does not** set `completedAt`. The comment at [`lib/utils/task-logger.ts#L105`](repo://lib/utils/task-logger.ts#L105) is explicit: *"completedAt is only set when PR is merged, not when status changes to 'completed'."* This makes "task completed" and "PR merged" distinguishable on the row.

## The logs column

`tasks.logs` is a JSONB array of `LogEntry` objects:

```ts
export const logEntrySchema = z.object({
  type: z.enum(['info', 'command', 'error', 'success']),
  message: z.string(),
  timestamp: z.date().optional(),
})
```

([`lib/db/schema.ts#L5-L9`](repo://lib/db/schema.ts#L5-L9))

The only writer from inside the worker is `TaskLogger` ([`lib/utils/task-logger.ts`](repo://lib/utils/task-logger.ts#L1-L139)), exposed through `createTaskLogger(taskId)`. Its shape:

- **`append(type, message)`** — reads the current `tasks.logs`, appends one entry, writes the full array back. This is a read-then-write inside a single helper call; concurrent writers from the same serverless execution can race, but in practice only the worker's own callbacks write logs. Errors are **swallowed** — the comment at [`task-logger.ts#L51`](repo://lib/utils/task-logger.ts#L51) is explicit: *"Don't throw — we don't want logging failures to break the main process."*
- **`info(message)` / `command(message)` / `error(message)` / `success(message)`** — typed convenience wrappers.
- **`updateProgress(progress, message)`** — atomic-feeling (in practice still a read-then-write) update of `progress` and append of an `info` log entry.
- **`updateStatus(status, message?)`** — accepts `pending | processing | completed | error` (no `stopped`) and optionally appends an `info` log entry in the same call.

Each log entry is created by `createInfoLog`/`createCommandLog`/`createErrorLog`/`createSuccessLog` in [`lib/utils/logging.ts#L76-L99`](repo://lib/utils/logging.ts#L76-L99), all of which run `redactSensitiveInfo(message)` before persisting — see the redaction page for what is scrubbed.

Streaming agents (`claude` etc.) bypass the log column for incremental output and instead update a single `taskMessages` row identified by `agentMessageId` in place. The `tasks.logs` column is therefore a coarse event stream ("phase started", "sandbox created", "agent finished"), not a token-level transcript.

## Soft delete vs. `status='stopped'`

These two fields look like they do the same thing but they answer different questions, and confusing them produces real bugs. The split is enforced both at the API layer and by every read query in the system.

| Aspect | `status='stopped'` | `deletedAt != NULL` |
| --- | --- | --- |
| **Meaning** | The task ran but was cancelled before completion | The user removed it from their view |
| **Set by** | `PATCH /api/tasks/:taskId` with `action: 'stop'` only ([`app/api/tasks/[taskId]/route.ts#L75-L84`](repo://app/api/tasks/[taskId]/route.ts#L75-L84)) | `DELETE /api/tasks/:taskId` only ([`app/api/tasks/[taskId]/route.ts#L139-L143`](repo://app/api/tasks/[taskId]/route.ts#L139-L143)) |
| **Row exists** | Yes, fully readable | Yes, fully present in the table |
| **Visible in `GET /api/tasks`** | Yes (lists all non-`deletedAt` rows) | **No** — every task-scoped read adds `isNull(tasks.deletedAt)` |
| **Visible in `GET /api/tasks/:taskId`** | Yes | No — returns `404 Task not found` |
| **Counted by `checkRateLimit`** | Yes (it counts `tasksToday` ignoring only `deletedAt`) | No (`isNull(tasks.deletedAt)` is in the WHERE clause) |
| **Worker still touches it?** | The worker reads `status` at `isTaskStopped` checkpoints and exits; the row is otherwise dormant | The worker is not involved |
| **User can still resume it?** | Yes — `POST /api/tasks/:taskId/continue` rewinds any non-pending row | No — the continue handler returns `404` |

The rationale is operational: a stopped task still has a `branchName`, conversation history, and (potentially) a live sandbox via `sandboxId`. Hiding it under `deletedAt` would lose all of that, while hiding a stopped task under `status='stopped'` would conflict with the natural meaning of "stopped" (the work was cancelled, not removed from view).

The bulk endpoint uses the **opposite** model: `DELETE /api/tasks?action=completed,failed,stopped` ([`app/api/tasks/route.ts#L738-L815`](repo://app/api/tasks/route.ts#L738-L815)) runs a real SQL `DELETE FROM tasks WHERE status IN (...) AND userId = ?` and returns the deleted rows. Bulk-delete is destructive and irreversible — there is no `deletedAt` tombstone — and it is the right tool for "Clear completed" / "Clear failed" UI buttons.

### Read-path filter

Every endpoint that addresses a single task includes the same three-clause filter:

```ts
where(and(
  eq(tasks.id, taskId),
  eq(tasks.userId, session.user.id),
  isNull(tasks.deletedAt),
))
```

Examples: `GET /api/tasks/:taskId` ([L22-L27](repo://app/api/tasks/[taskId]/route.ts#L22-L27)), `PATCH /api/tasks/:taskId` ([L51-L55](repo://app/api/tasks/[taskId]/route.ts#L51-L55)), `DELETE /api/tasks/:taskId` ([L129-L133](repo://app/api/tasks/[taskId]/route.ts#L129-L133)), `POST /:taskId/continue` ([L53-L57](repo://app/api/tasks/[taskId]/continue/route.ts#L53-L57)), `POST /:taskId/merge-pr` ([L28-L31](repo://app/api/tasks/[taskId]/merge-pr/route.ts#L28-L31)), `GET /api/tasks` ([L35-L39](repo://app/api/tasks/route.ts#L35-L39)). The same filter also gates the rate-limit counts.

## Rate limit (daily ceiling)

`POST /api/tasks` and `POST /api/tasks/:taskId/continue` both call `checkRateLimit(session.user.id)` ([`lib/utils/rate-limit.ts`](repo://lib/utils/rate-limit.ts#L6-L50)) before doing any work. A `429` is returned with `remaining`, `total`, and `resetAt` if the user has hit their daily cap.

The ceiling is a *combined* count, not a per-endpoint count:

```ts
const count = tasksToday.length + userMessagesToday.length
const allowed = count < maxMessagesPerDay
```

where:

- **`tasksToday`** — rows in `tasks` for this user with `createdAt >= startOfTodayUtc()` and `isNull(deletedAt)`. Soft-deleted rows are excluded, so deleting a task does not free a slot.
- **`userMessagesToday`** — rows in `taskMessages` (inner-joined to `tasks` to enforce ownership and `isNull(deletedAt)`) with `role='user'` and `createdAt >= startOfTodayUtc()`. Agent responses do not count.

The window is the **UTC day** — `today.setUTCHours(0, 0, 0, 0)` — not the user's local day, and `resetAt` is the next UTC midnight. The limit resolves through [`getMaxMessagesPerDay`](repo://lib/db/settings.ts#L57-L60): per-user `settings` row → `MAX_MESSAGES_PER_DAY` env → constant default `5` ([`lib/constants.ts#L2`](repo://lib/constants.ts#L2)). A user who has used `3` task creations and `2` follow-up messages today has `0` remaining; a `POST /api/tasks` returns `429` until the next UTC midnight.

The 429 body is the same shape on both endpoints:

```json
{
  "error": "Rate limit exceeded",
  "message": "You have reached the daily limit of 5 messages (tasks + follow-ups). Your limit will reset at 2026-09-15T00:00:00.000Z",
  "remaining": 0,
  "total": 5,
  "resetAt": "2026-09-15T00:00:00.000Z"
}
```

## API entrypoints and the guards they apply

The four task-scoped HTTP entrypoints apply different guards. All of them require a session via `getServerSession()`.

### `POST /api/tasks` — create

Order of operations ([`app/api/tasks/route.ts#L48-L250`](repo://app/api/tasks/route.ts#L48-L250)):

1. `getServerSession()` → `401` if no user.
2. `checkRateLimit(userId)` → `429` if over the daily cap (see above).
3. Body validation via `insertTaskSchema.parse({ ..., id, userId, status: 'pending', progress: 0, logs: [] })` ([`lib/db/schema.ts#L117-L147`](repo://lib/db/schema.ts#L117-L147)). The `id` is `body.id ?? generateId(12)`.
4. Insert; capture `newTask`.
5. **Snapshot credentials before `after()`**: `getUserApiKeys`, `getUserGitHubToken`, `getGitHubUser`, `getMaxSandboxDuration`. These cannot be read inside the `after()` callback because cookies are gone there.
6. Schedule three `after()` callbacks:
   - `generateBranchName` → AI Gateway if `AI_GATEWAY_API_KEY` is set, else skip; on failure writes `createFallbackBranchName(taskId)`.
   - `generateTaskTitle` → same shape, fallback `createFallbackTitle(prompt)`.
   - `processTaskWithTimeout(newTask.id, ...)` — the actual worker. The inline comment is *"Wrap in after() to ensure Vercel doesn't kill the function after response."*
7. Return `200 { task: newTask }`.

### `GET /api/tasks` and `GET /api/tasks/:taskId`

Both apply the standard `userId` + `isNull(deletedAt)` filter. `GET /api/tasks` returns rows in `desc(createdAt)` order ([L38-L39](repo://app/api/tasks/route.ts#L38-L39)).

### `PATCH /api/tasks/:taskId` with `action: 'stop'`

The only valid action today ([`app/api/tasks/[taskId]/route.ts#L40-L117`](repo://app/api/tasks/[taskId]/route.ts#L40-L117)). The guards are:

- **Status guard**: rejects with `400 Task can only be stopped when it is in progress` unless `existingTask.status === 'processing'`. The rationale is that there is nothing to stop for a `pending`, `completed`, `error`, or already-`stopped` row — the worker is either not yet started, already done, or already unwinding. This is the only API mutation gated by current `status`.
- **Mutation**: writes `status='stopped'`, `error='Task was stopped by user'`, `updatedAt=now`, `completedAt=now`.
- **Sandbox kill**: calls `killSandbox(taskId)` from [`lib/sandbox/sandbox-registry.ts#L25-L60`](repo://lib/sandbox/sandbox-registry.ts#L25-L60). The function pops the entry from the in-memory `Map<taskId, Sandbox>` and calls `sandbox.stop()`. It has a fallback: if no entry for `taskId` exists but the map is non-empty (e.g. a "Try Again" created a new id), it kills the oldest registered sandbox.
- **Logging**: `logger.info('Stop request received...')`, then on success `logger.success('Sandbox killed successfully')` or `logger.error('Failed to kill sandbox')`, then `logger.error('Task execution stopped by user')`. The stop path's log entries end with two `error` entries even on success — that is intentional, the design treats stop as an abnormal termination.

The PATCH response returns `200 { message: 'Task stopped successfully', task: updatedTask }` immediately. The worker discovers the status change at its next `isTaskStopped` checkpoint and exits without re-mutating state.

### `POST /api/tasks/:taskId/continue`

The rewind path for any non-pending row ([`app/api/tasks/[taskId]/continue/route.ts#L22-L116`](repo://app/api/tasks/[taskId]/continue/route.ts#L22-L116)):

1. Session and rate-limit checks.
2. Body validation: `message` required, non-empty after `.trim()`.
3. Task lookup with the standard filter; `404` if missing or `deletedAt` is set; `400 'Task does not have a branch to continue from'` if `branchName` is null.
4. Insert the user's message into `taskMessages` (`role='user'`).
5. **Reset the row**: `UPDATE tasks SET status='processing', progress=0, completedAt=NULL, updatedAt=now WHERE id = taskId`.
6. Snapshot credentials; `after(() => continueTask(...))`.

`continueTask` then either reconnects to the existing sandbox via `Sandbox.get(sandboxId)` (only when `task.keepAlive && task.sandboxId`) or builds a fresh sandbox with `preDeterminedBranchName: branchName`. The worker's terminal state writes are the same as for an initial run — `status='completed'` on push success, `status='error'` on any failure.

### `DELETE /api/tasks/:taskId`

Soft delete ([`app/api/tasks/[taskId]/route.ts#L119-L150`](repo://app/api/tasks/[taskId]/route.ts#L119-L150)). Sets `deletedAt = now()` and returns `200 { message: 'Task deleted successfully' }`. The row remains in the table; subsequent reads with `isNull(tasks.deletedAt)` filter return `404`.

### `DELETE /api/tasks?action=completed,failed,stopped`

Hard delete filtered by status ([`app/api/tasks/route.ts#L738-L815`](repo://app/api/tasks/route.ts#L738-L815)). Validates the `action` query parameter against `['completed','failed','stopped']`; rejects unknown values with `400`. Builds an `OR(...)` over the requested statuses, filters by `userId`, and runs `db.delete(tasks).where(whereClause).returning()`. The response includes a per-status count and a total. This is the only place in the codebase that hard-deletes rows from `tasks`.

## Invariants and failure modes

- **`status` is never null.** The Drizzle schema has `.notNull().default('pending')`, so even a misconfigured INSERT leaves a readable row.
- **The worker only ever writes four of the five statuses.** `TaskLogger.updateStatus` does not accept `'stopped'` ([`lib/utils/task-logger.ts#L107`](repo://lib/utils/task-logger.ts#L107)). The only writer of `'stopped'` is the PATCH handler.
- **Soft-deleted rows are invisible to every read path.** A `deletedAt != NULL` row is excluded from `GET /api/tasks`, `GET /:taskId`, all `PATCH`/`DELETE`/`POST continue`/`POST merge-pr` handlers, and the rate-limit counts. It is **not** deleted from the database, so any downstream system that bypasses the standard filter (e.g. an analytics query) will still see it.
- **`completedAt` is meaningful.** `null` means the task either never reached `completed` or was reanimated by a follow-up; non-null means either user-stopped or PR-merged. The absence of `completedAt` together with `status='completed'` is the expected state for a task that finished pushing its branch but whose PR has not yet been merged.
- **Stop is one-shot.** A stopped task cannot be stopped again (the PATCH status guard rejects with `400`), and a stopped task cannot be revived through the stop path. The only way forward from `stopped` is `POST /:taskId/continue` (which rewinds to `processing`).
- **Rate-limit reset is UTC.** A user in `America/Los_Angeles` will see their cap reset at 4 PM or 5 PM local time depending on DST. Code that surfaces the limit to users should format `resetAt` in the user's locale, not as UTC.
- **The `after()` worker cannot read the session.** Anything the worker needs from cookies, OAuth state, or request-scoped storage must be resolved before `processTaskWithTimeout(...)` is called inside `after(...)`. The error path inside the worker logs via `TaskLogger` only; there is no client to return to.
- **Logging is best-effort.** `TaskLogger` swallows DB errors to avoid masking real failures with log-append failures ([`task-logger.ts#L51-L54`](repo://lib/utils/task-logger.ts#L51-L54)). This means a `logs` array can be silently incomplete; do not rely on its presence for correctness.

## Extension points

- **New status value.** Add it to the Drizzle enum ([`lib/db/schema.ts#L90-L94`](repo://lib/db/schema.ts#L90-L94)) and to both `insertTaskSchema`/`selectTaskSchema`. Update `TaskLogger.updateStatus` if the worker should be allowed to write it. Add the transition to the state diagram above and to the bulk-delete `validActions` array in [`app/api/tasks/route.ts#L754`](repo://app/api/tasks/route.ts#L754) if users should be able to clear tasks in that state.
- **New rate-limit dimension.** Today the limit is a single `tasksToday + userMessagesToday` counter against one `maxMessagesPerDay`. A per-task-type or per-agent ceiling would need a new key in `settings` plus a parallel `getNumericSetting` lookup in [`lib/utils/rate-limit.ts`](repo://lib/utils/rate-limit.ts#L6-L50). Both `POST /api/tasks` and `POST /:taskId/continue` would need to call it.
- **New task-scoped endpoint.** Reuse the standard `and(eq(tasks.id, taskId), eq(tasks.userId, session.user.id), isNull(tasks.deletedAt))` filter so soft-deleted rows stay hidden. If the endpoint must mutate `status`, decide explicitly whether the worker, the API handler, or both should be the writer, and update the state diagram accordingly.
- **Streaming logs.** Today `TaskLogger` writes one row per call. For high-frequency updates (e.g. agent token streams) the existing pattern is to update a single `taskMessages` row in place identified by `agentMessageId` — see [`lib/sandbox/agents/claude.ts`](repo://lib/sandbox/agents/claude.ts) for the `Writable` stream that does this. A future change that wants richer `tasks.logs` history should batch writes and only flush on phase boundaries.
