---
type: workflow
title: Create and Run a Task
description: End-to-end happy path from the user typing a prompt into the task form, through POST /api/tasks persisting a pending row, three parallel next/server after() callbacks (AI branch name, AI title, and the worker), createSandbox spawning a Vercel Sandbox, executeAgentInSandbox dispatching to a CLI, pushChangesToBranch pushing the commit, and the React client polling GET /api/tasks/:taskId to render the live status.
tags: [workflow, task, sandbox, agent, after, polling, ai-gateway, vercel-sandbox, post-api-tasks, use-task]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-7ba88dffbf982709a6072164
    resource: repo://app/api/tasks/%5BtaskId%5D/route.ts
  - id: openwiki-source-c7baae88c7e2880cc027c016
    resource: repo://app/api/tasks/route.ts
  - id: openwiki-source-f306e4202309b8fb938e3e4d
    resource: repo://components/app-layout.tsx
  - id: openwiki-source-7b713bd3d987acffddc4c106
    resource: repo://components/home-page-content.tsx
  - id: openwiki-source-ceec82605dbc3ef332b3c1ed
    resource: repo://components/task-form.tsx
  - id: openwiki-source-526feb2334febba699169a23
    resource: repo://lib/db/schema.ts
  - id: openwiki-source-e0723004f9fa18ccde4425ab
    resource: repo://lib/hooks/use-task.ts
  - id: openwiki-source-f111cb78e7ed9c79f1238982
    resource: repo://lib/sandbox/agents/index.ts
  - id: openwiki-source-76eb5082498d555b89ce9862
    resource: repo://lib/sandbox/creation.ts
  - id: openwiki-source-8fa4c22275829a090314c345
    resource: repo://lib/sandbox/git.ts
  - id: openwiki-source-b4de2cd9d50e247e61519a30
    resource: repo://lib/utils/branch-name-generator.ts
  - id: openwiki-source-7ece006b3c5e6a3e70e9b390
    resource: repo://lib/utils/rate-limit.ts
  - id: openwiki-source-fa62636f4bcd66f2e663943b
    resource: repo://lib/utils/title-generator.ts
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Create and Run a Task

This page follows a single coding-agent run from the moment a user types a prompt in the home-page form to the moment the work is committed and pushed to a GitHub branch. It is the workflow complement to [Runtime Flow: Request → Sandbox → Agent → Push](../architecture/runtime-flow.md) (which exhaustively describes each phase), [Sandbox Lifecycle](../concepts/sandbox-lifecycle.md) (the in-VM phases), [Task Lifecycle & Status Model](../concepts/tasks-lifecycle.md) (the row-level state machine), and [Task API Surface](../systems/task-api.md) (the route catalog).

The happy path is the only path that returns a `200 { task }` to the client before the work is done — every other path (a rate-limit rejection, a sandbox creation failure, an agent failure, a push failure, a user stop) is a variation on this same skeleton. The shape that holds everything together is the **`after()` callback**: the HTTP handler returns within milliseconds, and three independent `after()` callbacks run after the response is sent, each owning a distinct slice of the work.

## Overview

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
sequenceDiagram
    autonumber
    participant U as User
    participant TF as TaskForm<br/>components/task-form.tsx
    participant HPC as HomePageClient<br/>home-page-content.tsx
    participant AL as AppLayout<br/>app-layout.tsx
    participant API as POST /api/tasks<br/>app/api/tasks/route.ts
    participant DB as Postgres (tasks)
    participant A1 as after() #1<br/>generateBranchName
    participant A2 as after() #2<br/>generateTaskTitle
    participant A3 as after() #3<br/>processTaskWithTimeout
    participant AG as AI Gateway<br/>vercel.com/docs/ai-gateway
    participant SB as createSandbox<br/>lib/sandbox/creation.ts
    participant VS as Vercel Sandbox SDK
    participant AG2 as executeAgentInSandbox<br/>lib/sandbox/agents/index.ts
    participant CLI as Agent CLI in sandbox
    participant GIT as pushChangesToBranch<br/>lib/sandbox/git.ts
    participant TL as TaskLogger<br/>lib/utils/task-logger.ts
    participant TPC as TaskPageClient<br/>task-page-client.tsx
    participant UT as useTask hook<br/>lib/hooks/use-task.ts
    participant POLL as GET /api/tasks/:taskId

    U->>TF: type prompt + select agent/model/repo + click submit
    TF->>HPC: onSubmit({ prompt, repoUrl, selectedAgent, ... })
    HPC->>AL: addTaskOptimistically(nanoid) -> id
    HPC->>HPC: router.push(/tasks/{id}) immediately
    HPC->>API: fetch POST /api/tasks body={...data, id}
    API->>API: getServerSession() (401 if missing)
    API->>API: checkRateLimit(userId) (429 if over daily cap)
    API->>API: insertTaskSchema.parse(body) (400 if invalid)
    API->>DB: INSERT tasks (status=pending, progress=0, logs=[])
    DB-->>API: newTask row

    par Parallel enrichment after the response is sent
        API->>A1: after() -> generateBranchName
        A1->>AG: generateText openai/gpt-5-nano
        AG-->>A1: base name
        A1->>DB: UPDATE branchName (or fallback)
    and
        API->>A2: after() -> generateTaskTitle
        A2->>AG: generateText openai/gpt-5-nano
        AG-->>A2: title
        A2->>DB: UPDATE title (or fallback)
    end

    API->>A3: after() -> processTaskWithTimeout(...)
    API-->>HPC: 200 { task: newTask }
    HPC->>TPC: navigate to /tasks/{id}

    Note over A3,POLL: while the worker runs, the client polls every 5s

    A3->>DB: getUserApiKeys / getUserGitHubToken / getGitHubUser (snapshot)
    A3->>A3: processTaskWithTimeout -> processTask
    A3->>TL: updateStatus(processing) + updateProgress(10)
    A3->>DB: INSERT taskMessages (role=user)
    A3->>DB: isTaskStopped check (early)
    A3->>DB: waitForBranchName (poll <=10s)
    A3->>DB: isTaskStopped check (after branch gen)

    A3->>SB: createSandbox(SandboxConfig)
    SB->>VS: Sandbox.create(timeout, ports, runtime)
    SB->>VS: mkdir PROJECT_DIR + git clone --depth 1
    SB->>VS: installDependencies + start dev server
    SB->>VS: git config user.name/email + resolve branch
    SB-->>A3: { sandbox, domain, branchName }

    A3->>DB: UPDATE sandboxId / sandboxUrl / branchName
    A3->>DB: isTaskStopped check (before agent)
    A3->>DB: load connected MCP connectors + decrypt
    A3->>AG2: executeAgentInSandbox(sandbox, prompt, agentType, ...)
    AG2->>CLI: install + run claude/codex/copilot/cursor/gemini/opencode
    CLI-->>AG2: AgentExecutionResult
    AG2-->>A3: { success, sessionId, agentResponse }
    A3->>DB: persist agentSessionId
    A3->>DB: INSERT taskMessages (role=agent)
    A3->>AG: generateCommitMessage (AI Gateway or fallback)
    A3->>GIT: pushChangesToBranch(sandbox, branch, message)
    GIT->>VS: git status / add / commit / push origin branch
    GIT-->>A3: { success, pushFailed? }

    alt push succeeded and keepAlive=false
        A3->>VS: shutdownSandbox (pkill + unregister)
        A3->>TL: updateStatus(completed) + updateProgress(100)
    else push failed
        A3->>TL: updateStatus(error)
    end

    Note over TPC,POLL: meanwhile the client sees the row update
    UT->>POLL: fetch /api/tasks/{taskId} (every 5s)
    POLL-->>UT: { task: <updated row> }
    UT->>TPC: setTask(<updated row>)
    TPC->>TPC: render TaskDetails + LogsPane with live status, branchName, sandboxUrl, logs
```

*Diagram: the happy path from form submit to branch push. The HTTP response returns within milliseconds; the three `after()` callbacks then run in parallel after the response is sent, with the worker driving the sandbox/agent/push pipeline while the client polls every five seconds for status updates.*

## Step 1 — User fills the form

The entry point is `TaskForm` in [`components/task-form.tsx`](repo://components/task-form.tsx). It is a controlled form with three regions:

- **Prompt textarea** (`task-form.tsx#L428-L439`) — a `Textarea` bound to the global Jotai atom `taskPromptAtom` so the prompt survives navigation. `Enter` on desktop submits the form via `form.dispatchEvent(new Event('submit', ...))`; on mobile, `Enter` inserts a newline and the user must tap the submit button.
- **Agent + model select** (`task-form.tsx#L448-L555`) — the agent dropdown enumerates `claude`, `codex`, `copilot`, `cursor`, `gemini`, `opencode`, plus a `multi-agent` "Compare" option that lets the user pick one model per agent. Per-agent model lists live in `AGENT_MODELS` (`task-form.tsx#L73-L121`); per-agent default models live in `DEFAULT_MODELS` (`task-form.tsx#L124-L131`). The selected values are persisted to two Jotai atoms (`lastSelectedAgentAtom` and `lastSelectedModelAtomFamily(selectedAgent)`) so the form restores them on remount.
- **Option chips / dropdown** (`task-form.tsx#L557-L756`) — three booleans plus one numeric:
  - `installDependencies` — whether `createSandbox` should run `npm install` / `pnpm install` / `pip install`. Defaults to `false`.
  - `maxDuration` — sandbox lifetime in minutes, one of `5/10/15/30/45/60/120/180/240/300`. Defaults to the user's `maxSandboxDuration` setting (clamped server-side by `getMaxSandboxDuration`).
  - `keepAlive` — when `true`, the sandbox is left running after `pushChangesToBranch` so a follow-up message can reconnect via `Sandbox.get(sandboxId)`. Does **not** extend the SDK timeout; see [Sandbox Lifecycle → The `keepAlive` flag](../concepts/sandbox-lifecycle.md#the-keepalive-flag).
  - `enableBrowser` — when `true`, `createSandbox` installs Chromium + the `agent-browser` CLI plus a per-agent skill file describing it. Used by agents that drive a headless browser.

`handleSubmit` ([`task-form.tsx#L325-L396`](repo://components/task-form.tsx#L325-L396)) does three things in order before calling the parent's `onSubmit`:

1. Trims the prompt and bails if empty.
2. For `multi-agent` mode, requires at least one model in `selectedModels`.
3. Hits `/api/api-keys/check?agent=...&model=...` to confirm the required provider key is present, and toasts an error if it is not (the user's settings dialog is where keys are added; this is the only client-side guard against submitting a task that will fail at env validation inside `createSandbox`).

The component is wrapped by `HomePageContent` in [`components/home-page-content.tsx`](repo://components/home-page-content.tsx), which passes `handleTaskSubmit` as `onSubmit` (see [`home-page-content.tsx#L563`](repo://components/home-page-content.tsx#L563)).

## Step 2 — Optimistic insert + POST `/api/tasks`

`handleTaskSubmit` in [`home-page-content.tsx#L329-L543`](repo://components/home-page-content.tsx#L329-L543) is the client-side driver. It runs **three** branches depending on the form state:

- **Single agent + single repo** (the happy path described here): `addTaskOptimistically(data)` creates a temporary row with a `nanoid` and `status='pending'` ([`app-layout.tsx#L210-L255`](repo://components/app-layout.tsx#L210-L255)), inserts it at the head of the sidebar tasks array, and returns the generated `id`. The handler then `router.push('/tasks/<id>')` immediately and fires `fetch('/api/tasks', { method: 'POST', body: JSON.stringify({ ...data, id }) })`. Including the pre-generated `id` in the body means the server can `INSERT` with that exact `id` and the optimistic row in the sidebar will reconcile to the same identity once the response returns.
- **Multi-repo** ([`home-page-content.tsx#L370-L437`](repo://components/home-page-content.tsx#L370-L437)): one optimistic row per selected repo, one `POST /api/tasks` per row in parallel, then `router.push` to the first task id.
- **Multi-agent** ([`home-page-content.tsx#L442-L506`](repo://components/home-page-content.tsx#L442-L506)): parses each `agent:model` string into `{ selectedAgent, selectedModel }`, one optimistic row per model, one `POST /api/tasks` per row in parallel.

In every branch the navigation happens **before** the POST returns. This is what makes the UI feel instant: by the time the server is still validating the request, the user is already on `/tasks/<id>` staring at the live `useTask` polling state.

While that happens, the server-side handler runs.

## Step 3 — `POST /api/tasks` (synchronous, returns within milliseconds)

[`app/api/tasks/route.ts#L48-L250`](repo://app/api/tasks/route.ts#L48-L250) is a thin shell. It does six things in order, all before sending the response:

1. **Authenticate.** `await getServerSession()`. `401 { error: 'Unauthorized' }` if there is no session.
2. **Rate-limit.** `await checkRateLimit(session.user.id)` (see [`lib/utils/rate-limit.ts`](repo://lib/utils/rate-limit.ts)). Counts both `tasksToday` and `userMessagesToday` (join through `taskMessages.role='user'`), enforces the per-user `maxMessagesPerDay` cap (user setting > global setting > env var). `429 { error, message, remaining, total, resetAt }` with the daily total and UTC reset time when over the cap.
3. **Validate.** `insertTaskSchema.parse({ ...body, id: taskId, userId, status: 'pending', progress: 0, logs: [] })`. `taskId` is `body.id || generateId(12)` so a client-provided id is honored (this is what the optimistic insert relies on). On a Zod failure, Zod throws and the catch-all at the bottom returns `500 { error: 'Failed to create task' }`.
4. **Insert.** `db.insert(tasks).values({ ...validatedData, id: taskId }).returning()` yields `newTask`.
5. **Snapshot credentials.** `getUserApiKeys()`, `getUserGitHubToken()`, `getGitHubUser()`, and `getMaxSandboxDuration(session.user.id)`. This must happen **before** `after()` runs because cookies are not available inside the background callback — see Step 4 below.
6. **Return.** `NextResponse.json({ task: newTask })` with `status='pending'`.

The handler does **not** start the worker itself. Instead it schedules three `after()` callbacks and returns. The three callbacks are the heart of the workflow.

## Step 4 — Three parallel `after()` callbacks

[`after()` from `next/server`](repo://app/api/tasks/route.ts#L1) is the Next.js primitive that schedules a callback to run *after* the response has been flushed but before the serverless instance is frozen. It is the only reason a multi-minute task can live inside a route handler. The source comment at [`route.ts#L221-L222`](repo://app/api/tasks/route.ts#L221-L222) is explicit:

> CRITICAL: Wrap in after() to ensure Vercel doesn't kill the function after response. Without this, serverless functions terminate immediately after sending the response.

Two practical consequences flow from this:

- **All session-scoped I/O must be resolved before `after()`.** `getServerSession`, `getUserApiKeys`, `getUserGitHubToken`, `getGitHubUser` all run synchronously above the `after()` calls. Inside the callback the cookies are gone.
- **Errors thrown inside `after()` are not surfaced to the client.** They are logged with `console.error` and the `TaskLogger` is responsible for persisting them to the `tasks` row.

### The two parallel enrichment callbacks

These two callbacks fire concurrently and each writes back a single column to the row:

```ts
// after() #1 — branch name via AI Gateway
after(async () => {
  if (!process.env.AI_GATEWAY_API_KEY) {
    console.log('AI_GATEWAY_API_KEY not available, skipping AI branch name generation')
    return
  }
  const logger = createTaskLogger(taskId)
  await logger.info('Generating AI-powered branch name...')
  // ... extract repoName from validatedData.repoUrl ...
  const aiBranchName = await generateBranchName({
    description: validatedData.prompt,
    repoName,
    context: `${validatedData.selectedAgent} agent task`,
  })
  await db.update(tasks).set({ branchName: aiBranchName, updatedAt: new Date() }).where(eq(tasks.id, taskId))
  await logger.success('Generated AI branch name')
}) // route.ts#L94-L155

// after() #2 — title via AI Gateway
after(async () => {
  if (!process.env.AI_GATEWAY_API_KEY) {
    console.log('AI_GATEWAY_API_KEY not available, skipping AI title generation')
    return
  }
  // ... extract repoName ...
  const aiTitle = await generateTaskTitle({
    prompt: validatedData.prompt,
    repoName,
    context: `${validatedData.selectedAgent} agent task`,
  })
  await db.update(tasks).set({ title: aiTitle, updatedAt: new Date() }).where(eq(tasks.id, taskId))
}) // route.ts#L158-L211
```

Both helpers live under [`lib/utils/`](repo://lib/utils/):

- [`generateBranchName`](repo://lib/utils/branch-name-generator.ts#L10-L71) calls `generateText` from `ai` with model `openai/gpt-5-nano` through Vercel AI Gateway, validates the result against `^[a-z0-9-\/]+$`, caps the total length at 50 characters, and appends a 6-character alphanumeric hash to avoid collisions. The full branch name has the shape `<base>-<hash>`. On any failure the catch block writes `createFallbackBranchName(taskId)` ([`branch-name-generator.ts#L73-L76`](repo://lib/utils/branch-name-generator.ts#L73-L76)) which is `agent/<iso-timestamp>-<id8>`.
- [`generateTaskTitle`](repo://lib/utils/title-generator.ts#L9-L60) calls the same gateway with the same model and writes a sentence-cased title capped at 60 characters (`title.substring(0, 57) + '...'` on overflow). Its fallback is `createFallbackTitle(prompt)` ([`title-generator.ts#L62-L70`](repo://lib/utils/title-generator.ts#L62-L70)) which uses the trimmed prompt verbatim when it is short enough.

The `AI_GATEWAY_API_KEY` early-return is important: deployments without an AI Gateway key still create the task, the row simply has a null `title` / `branchName` until the sandbox-creation step (see Step 6) supplies the fallback.

### The worker callback — `processTaskWithTimeout`

This is the third `after()` and the only one that drives the sandbox/agent/push pipeline:

```ts
after(async () => {
  try {
    await processTaskWithTimeout(
      newTask.id,
      validatedData.prompt,
      validatedData.repoUrl || '',
      validatedData.maxDuration || maxSandboxDuration,
      validatedData.selectedAgent || 'claude',
      validatedData.selectedModel,
      validatedData.installDependencies || false,
      validatedData.keepAlive || false,
      validatedData.enableBrowser || false,
      userApiKeys,
      userGithubToken,
      githubUser,
    )
  } catch (error) {
    console.error('Task processing failed:', error)
    // Error handling is already done inside processTaskWithTimeout
  }
}) // route.ts#L223-L243
```

Notice that the `userApiKeys`, `userGithubToken`, and `githubUser` snapshots from Step 3 are passed in as arguments. `processTaskWithTimeout` cannot call `getServerSession()` because there is no request scope inside `after()`.

## Step 5 — `processTaskWithTimeout` → `processTask`

[`processTaskWithTimeout`](repo://app/api/tasks/route.ts#L252-L332) is the timeout wrapper. It computes `TASK_TIMEOUT_MS = maxDuration * 60 * 1000`, schedules a one-minute-before-deadline warning log, and races `processTask(...)` against a `setTimeout` reject. The loser is handled:

- **Timeout wins.** `logger.error('Task execution timed out')` + `logger.updateStatus('error', 'Task execution timed out. ...')`. Both timers are cleared.
- **Worker wins.** Clear the warning timer. If the worker threw, `processTaskWithTimeout` rethrows; the outer `after()` catches it.

[`processTask`](repo://app/api/tasks/route.ts#L366-L736) is the actual pipeline. Its sequencing is the canonical happy path documented in [Runtime Flow: Request → Sandbox → Agent → Push](../architecture/runtime-flow.md). The condensed version is:

1. `logger.updateStatus('processing', 'Task created, preparing to start...')` and `logger.updateProgress(10, 'Initializing task execution...')`.
2. Insert the user's prompt into `taskMessages` (`role='user'`).
3. **First `isTaskStopped` checkpoint.** `SELECT status FROM tasks WHERE id=?`; if `status === 'stopped'`, log and return.
4. `waitForBranchName(taskId, 10000)` — poll for up to 10 seconds for the AI-generated branch name from `after()` #1 to land. **Second `isTaskStopped` checkpoint** fires after the wait. If the AI branch is not ready, the log line `"AI branch name not ready, will use fallback during sandbox creation"` is written and the sandbox-creation step supplies its own.
5. `logger.updateProgress(15, 'Creating sandbox environment')`, then `detectPortFromRepo(repoUrl, githubToken)` so the sandbox is created with the correct port exposed.
6. `createSandbox({ taskId, repoUrl, githubToken, gitAuthorName, gitAuthorEmail, apiKeys, timeout: '<maxDuration>m', ports: [port], runtime: 'node22', resources: { vcpus: 4 }, taskPrompt, selectedAgent, selectedModel, installDependencies, keepAlive, enableBrowser, preDeterminedBranchName, onProgress, onCancellationCheck }, logger)`.
7. On `sandboxResult.success`, write `sandboxId` + `sandboxUrl` + (only if the AI branch was not ready) `branchName` to the row at [`route.ts#L518`](repo://app/api/tasks/route.ts#L518). This is the moment the UI gains a `sandboxUrl` it can render in the preview iframe.
8. **Third `isTaskStopped` checkpoint** ([`route.ts#L521`](repo://app/api/tasks/route.ts#L521)). If stopped, `shutdownSandbox` and return.
9. Load connected MCP connectors for the current user, decrypt `env` and `oauthClientSecret`, persist `mcpServerIds` to the row.
10. Sanitize the prompt (strip backticks / `$` / `\`, prefix any `-`-leading line with a space).
11. Generate `agentMessageId` and call `executeAgentInSandbox(sandbox, sanitizedPrompt, selectedAgent, logger, selectedModel, mcpServers, undefined, apiKeys, undefined, undefined, taskId, agentMessageId)`.
12. Persist `agentResult.sessionId` to `agentSessionId`, insert the agent's response message into `taskMessages`, generate an AI commit message (or fallback), and call `pushChangesToBranch(sandbox, branchName, commitMessage, logger)`.
13. Branch on `keepAlive`:
    - `keepAlive=true` — log "Sandbox kept alive for follow-up messages" and leave the row's `sandboxId` / `sandboxUrl` intact.
    - `keepAlive=false` — `unregisterSandbox(taskId)` + `shutdownSandbox(sandbox)`. The SDK garbage-collects the VM after its timeout.
14. If `pushResult.pushFailed === true`, `logger.updateStatus('error')` + `logger.error('Task failed: Unable to push changes to repository')` + throw. Otherwise `logger.updateStatus('completed')` + `logger.updateProgress(100, 'Task completed successfully')`.

The full deep dive (including the five cancellation checkpoints inside this pipeline and the four inside `createSandbox`) is in [Runtime Flow](../architecture/runtime-flow.md). The role of the sandbox phases — clone, dependencies, dev server, browser install, git config, branch resolution — is in [Sandbox Lifecycle](../concepts/sandbox-lifecycle.md).

## Step 6 — `executeAgentInSandbox` (dispatcher)

[`executeAgentInSandbox`](repo://lib/sandbox/agents/index.ts#L18-L159) is a thin dispatcher. It does four things regardless of which agent is selected:

1. Run the optional `onCancellationCheck` (processTask does not pass one for the initial run; follow-up runs do).
2. Lazy-load the user's GitHub token via `await import('@/lib/github/user-token')` when `agentType === 'copilot'` — Copilot authenticates through the user's GitHub account rather than an API key.
3. Apply the **temp-env-var pattern** ([`agents/index.ts#L57-L75`](repo://lib/sandbox/agents/index.ts#L57-L75)). Snapshot the current values of `OPENAI_API_KEY`, `GEMINI_API_KEY`, `CURSOR_API_KEY`, `ANTHROPIC_API_KEY`, `AI_GATEWAY_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN` into `originalEnv`. Write the per-user API keys into `process.env`. The `try/finally` at [`agents/index.ts#L149-L158`](repo://lib/sandbox/agents/index.ts#L149-L158) restores the originals regardless of outcome — this is the boundary that prevents one user's keys from leaking into the next user's run.
4. Switch on `agentType` and delegate to the per-agent module:

| Agent | Module | Auth |
| --- | --- | --- |
| `claude` | `lib/sandbox/agents/claude.ts` | `ANTHROPIC_API_KEY` + `ai-gateway.vercel.sh` base URL |
| `codex` | `lib/sandbox/agents/codex.ts` | `AI_GATEWAY_API_KEY` as OpenAI proxy |
| `copilot` | `lib/sandbox/agents/copilot.ts` | User's GitHub token (lazy-loaded above) |
| `cursor` | `lib/sandbox/agents/cursor.ts` | `CURSOR_API_KEY` |
| `gemini` | `lib/sandbox/agents/gemini.ts` | `GEMINI_API_KEY` |
| `opencode` | `lib/sandbox/agents/opencode.ts` | `AI_GATEWAY_API_KEY` or `ANTHROPIC_API_KEY` (model-dependent) |

Each module exports `execute<Type>InSandbox(sandbox, instruction, logger, selectedModel, mcpServers, isResumed, sessionId, taskId?, agentMessageId?)` and is responsible for installing its CLI inside the sandbox, writing any per-agent config file, registering MCP servers (when `mcpServers.length > 0`), running the CLI in detached mode with `Writable` streams that update the `taskMessages` row identified by `agentMessageId` in real time, and returning `AgentExecutionResult`.

## Step 7 — `pushChangesToBranch` and DB finalization

[`pushChangesToBranch`](repo://lib/sandbox/git.ts#L5-L73) is the only place the sandbox does git work after the agent finishes. Its contract is `{ success: boolean; pushFailed?: boolean }`:

1. `git status --porcelain` — empty output means no changes, returns `{ success: true }` early.
2. `git add .` — failure returns `{ success: false }`.
3. `git commit -m <message>` — failure returns `{ success: false }`.
4. `git push origin <branch>` — success returns `{ success: true }`. A failure whose stderr contains `Permission`, `access_denied`, or `403` logs a permission hint and returns `{ success: true, pushFailed: true }` (the local commit still exists). Any other failure returns `{ success: true, pushFailed: true }` too — the caller decides what to do.

Back in `processTask`:

- **`pushFailed === false` and `keepAlive === false`** — `unregisterSandbox(taskId)` + `shutdownSandbox(sandbox)` (best-effort `pkill -f node|python|npm|yarn|pnpm`). Then `logger.updateStatus('completed')` + `logger.updateProgress(100, 'Task completed successfully')`. The task is done.
- **`pushFailed === false` and `keepAlive === true`** — log "Sandbox kept alive for follow-up messages" and return. `sandboxId` / `sandboxUrl` stay on the row; a follow-up message hits `POST /api/tasks/:taskId/continue` and reconnects via `Sandbox.get(sandboxId)`. This is the [Workflow: Continue & Keep Alive](./follow-up-and-keep-alive.md) entry point.
- **`pushFailed === true`** — `logger.updateStatus('error')` + `logger.error('Task failed: Unable to push changes to repository')` + throw. The outer catch sets `status='error'`.

The keep-alive flag also gates whether the sandbox is shut down on **error**: when `keepAlive=true`, the error branch ([`route.ts#L710-L727`](repo://app/api/tasks/route.ts#L710-L727)) skips `unregisterSandbox`/`shutdownSandbox` so the user can retry against the same VM. See [Sandbox Lifecycle → Path A](../concepts/sandbox-lifecycle.md#path-a--worker-initiated-shutdown-the-common-path) for the worker-initiated shutdown path.

## Step 8 — UI polling for status

While `processTask` runs, the client is on `/tasks/<id>` rendering the live state via `useTask`. The page composition is:

- [`app/tasks/[taskId]/page.tsx`](repo://app/tasks/[taskId]/page.tsx) — server component that resolves `session`, `maxSandboxDuration`, and `getGitHubStars`, then mounts `<TaskPageClient />`.
- [`components/task-page-client.tsx`](repo://components/task-page-client.tsx) — renders `SharedHeader`, `<TaskDetails />`, and `<LogsPane />`. While `isLoading` is true only the header is rendered; on `error || !task` it renders a "Task Not Found" state; otherwise the full task UI.
- [`lib/hooks/use-task.ts`](repo://lib/hooks/use-task.ts) — the polling state machine.

`useTask` has three layers of behavior:

1. **Initial fetch with retry.** When `taskId` is set the hook `fetchTask()`s `GET /api/tasks/<taskId>` immediately and then re-fetches every 2 seconds for up to 3 attempts until the row is found. This handles the race where the user navigates to `/tasks/<id>` before the server's `INSERT` has committed (the optimistic insert in the sidebar may have already added the row client-side, but the server's `GET` 404s until the DB row exists).
2. **Steady-state polling.** Once `isLoading` is `false`, the hook re-fetches every 5 seconds ([`use-task.ts#L77-L83`](repo://lib/hooks/use-task.ts#L77-L83)). The cadence is intentional: it is fast enough to feel live (sandbox creation, dev server startup, agent output all happen within seconds) and slow enough not to drown the database.
3. **Sandbox-URL acceleration.** When `task.logs` contains the line `"Development server is running"` (or `"Development server started"`) but `task.sandboxUrl` is still null, the hook schedules an immediate refetch after 500 ms ([`use-task.ts#L86-L103`](repo://lib/hooks/use-task.ts#L86-L103)). This short-circuits the 5-second wait for the moment the preview iframe is about to become useful.

Each `GET /api/tasks/:taskId` ([`app/api/tasks/[taskId]/route.ts#L15-L38`](repo://app/api/tasks/[taskId]/route.ts#L15-L38)) returns `{ task }` after the standard ownership + soft-delete filter. It is the only place the polling path reads; `useTask` does not subscribe to websockets, SSE, or Postgres LISTEN/NOTIFY.

## Configuration and operations

The happy path has four environmental dependencies that must be present before the user can submit a task:

| Env var | Purpose | Failure mode |
| --- | --- | --- |
| `SANDBOX_VERCEL_TEAM_ID` / `SANDBOX_VERCEL_PROJECT_ID` / `SANDBOX_VERCEL_TOKEN` | Identify the Vercel Sandbox project | `createSandbox` throws at env validation; task ends as `error` |
| `AI_GATEWAY_API_KEY` | Powers the branch-name, title, and commit-message generators | Each `after()` early-returns; fallbacks fill the row |
| `MAX_SANDBOX_DURATION` (default `300`) | Default `maxDuration` if the user has no setting | Falls back to `300` minutes; per-user setting overrides |
| `JWE_SECRET` / `ENCRYPTION_KEY` | Session cookie encryption + connector env/secrets | Sign-in or connector read fails before the task can start |

User-side overrides (per-user or per-task) take precedence: `getMaxSandboxDuration(userId)` returns `user-specific > global > env var` and `getMaxMessagesPerDay(userId)` follows the same chain.

The two privilege boundaries that gate submission are `getServerSession()` (no session → `401`) and `checkRateLimit(userId)` (over daily cap → `429`). The third gate is the Zod parse of `insertTaskSchema`; a malformed body becomes a `500` from the catch-all.

## Failure modes

The happy path can degrade at any of the following checkpoints; each has a well-defined recovery:

- **`POST /api/tasks` returns non-200** — `home-page-content.tsx#L527-L533` reads `error.message` or `error.error` from the response and shows a toast; the optimistic sidebar row stays until the next `refreshTasks()` reconciles it.
- **`processTask` throws** — the outer `try/catch` ([`route.ts#L706-L735`](repo://app/api/tasks/route.ts#L706-L735)) shuts down the sandbox (unless `keepAlive=true`), logs `"Error occurred during task processing"`, and writes `status='error'`. The DB row is the source of truth; the user sees the error via the next 5-second poll.
- **`processTaskWithTimeout` rejects with the timeout message** — `logger.updateStatus('error', 'Task execution timed out. ...')` fires from the timeout branch. Note: the SDK's sandbox timeout is independent and may garbage-collect the VM before the JS-side `Promise.race` rejects.
- **`pushChangesToBranch` reports `pushFailed: true`** — `status='error'` is set and the local commit remains in the sandbox. The user can either retry or open the sandbox terminal to inspect.
- **`isTaskStopped` checkpoint fires during execution** — the worker returns early and the row ends as `status='stopped'` (set by `PATCH /api/tasks/:taskId` with `action: 'stop'`). See [Runtime Flow → Cancellation checkpoints](../architecture/runtime-flow.md#cancellation-checkpoints-the-full-list) for the full list.
- **User-initiated stop after the worker has already finished** — `killSandbox(taskId)` finds no entry in the in-memory `Map` (the worker released it) and the SDK timeout bounds the VM's lifetime. The DB write of `status='stopped'` still happens; the practical effect is just the status flip.

## Cross-cutting concerns

- **`after()` is the linchpin.** Everything that runs longer than a single HTTP round-trip — AI enrichment, the worker itself, future follow-up work — is wrapped in `after()`. The comment at [`route.ts#L221-L222`](repo://app/api/tasks/route.ts#L221-L222) is the canonical statement of why.
- **Session-scoped I/O before `after()`.** All four pre-worker reads (`getUserApiKeys`, `getUserGitHubToken`, `getGitHubUser`, `getMaxSandboxDuration`) are resolved synchronously above the `after()` block. The worker cannot reach them from inside the callback, so they are passed as arguments.
- **The DB row is the source of truth.** Every state change the worker makes — `status`, `progress`, `logs`, `branchName`, `sandboxId`, `sandboxUrl`, `agentSessionId`, `mcpServerIds` — is written to the `tasks` row. The polling client reads from the same row. There is no in-memory coupling between the worker and the client.
- **Log redaction.** `TaskLogger` ([`lib/utils/task-logger.ts`](repo://lib/utils/task-logger.ts)) is the only writer of `tasks.logs`. Every `info/command/error/success/updateProgress/updateStatus` call appends a `LogEntry` to the JSONB column; failures inside the logger are swallowed so a logging failure cannot break the worker.
- **Optimistic UI.** The sidebar task list ([`components/app-layout.tsx#L210-L255`](repo://components/app-layout.tsx#L210-L255)) shows a `pending` row before the server has even committed, and the task page itself renders the optimistic `pending` row while `useTask` races the server. The pre-generated `id` passed in the request body is what lets the two converge on the same identity.
- **Sandbox URL ≠ task completion.** The user can navigate to the preview iframe as soon as `sandboxUrl` is set (somewhere around `logger.updateProgress(30, ...)` in `createSandbox`). The push and `status='completed'` come later, possibly minutes after. The `useTask` polling cadence is tuned so the sandbox URL is rendered within ~500 ms of it appearing on the row.

## What to read next

- [Runtime Flow: Request → Sandbox → Agent → Push](../architecture/runtime-flow.md) — the full phase-by-phase walkthrough, including all five cancellation checkpoints and the `Promise.race` timeout semantics.
- [Sandbox Lifecycle](../concepts/sandbox-lifecycle.md) — the in-VM phases (creation, cloning, dependencies, dev server, push, shutdown), the dual persistence model (`Map<taskId, Sandbox>` vs `tasks.sandboxId` / `tasks.sandboxUrl`), and what `keepAlive` does and does not control.
- [Task Lifecycle & Status Model](../concepts/tasks-lifecycle.md) — the row schema, the five statuses, when `completedAt` is written, and soft-delete semantics.
- [Task API Surface](../systems/task-api.md) — the full route catalog under `/api/tasks`, including `GET /api/tasks/:taskId` that backs `useTask`.
- [Connectors (MCP Servers)](../systems/connectors-mcp.md) — how the MCP connectors that `processTask` decrypts and passes to `executeAgentInSandbox` are added and managed.
