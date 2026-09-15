---
type: system
title: Agent Implementations
description: How the six per-CLI agent wrappers in lib/sandbox/agents/ (claude, codex, copilot, cursor, gemini, opencode) install their CLI, configure authentication, attach MCP servers, stream output back to the UI, capture a session ID for follow-up turns, and report an AgentExecutionResult.
tags: [agents, claude, codex, copilot, cursor, gemini, opencode, cli, install, auth, mcp, streaming, session-resume, sandbox]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-3401b854c53347910d2c84ce
    resource: repo://app/api/tasks/%5BtaskId%5D/continue/route.ts
  - id: openwiki-source-a02d819c42a405c6b6116e8f
    resource: repo://lib/sandbox/agents/claude.ts
  - id: openwiki-source-1f4c6b999a2fa110a4b14e5f
    resource: repo://lib/sandbox/agents/codex.ts
  - id: openwiki-source-5adc19616f446e2a3f22d17b
    resource: repo://lib/sandbox/agents/copilot.ts
  - id: openwiki-source-461c38d111a1a74442ab8847
    resource: repo://lib/sandbox/agents/cursor.ts
  - id: openwiki-source-7563e3ff5929a46c9db6c26a
    resource: repo://lib/sandbox/agents/gemini.ts
  - id: openwiki-source-f111cb78e7ed9c79f1238982
    resource: repo://lib/sandbox/agents/index.ts
  - id: openwiki-source-bbbc9cdf967cf52b393b8240
    resource: repo://lib/sandbox/agents/opencode.ts
  - id: openwiki-source-a72f2779111752ff37aba5c4
    resource: repo://lib/sandbox/commands.ts
  - id: openwiki-source-78237118aac1cf9686d70d4d
    resource: repo://lib/sandbox/config.ts
  - id: openwiki-source-ad606bc69911c08f128163fa
    resource: repo://lib/sandbox/types.ts
  - id: openwiki-source-4c0dfa7b4928caf4818e73f8
    resource: repo://lib/utils/logging.ts
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Agent Implementations

Each supported coding-agent CLI has a dedicated wrapper module in `lib/sandbox/agents/` that owns every step between "the sandbox exists" and "the agent run produced an `AgentExecutionResult`". The dispatcher in `lib/sandbox/agents/index.ts` routes by the `AgentType` union; the wrappers themselves choose how to install the CLI, how to expose credentials, how to attach MCP servers, how to stream output, and how to extract a session ID for a follow-up turn.

This page documents what is common to every wrapper (the contract, the auth requirement, the result shape) and what differs (the install command, the MCP config dialect, whether the agent streams, how it resumes). For the dispatcher's responsibilities — the `apiKeys` overlay, the `finally`-restored `process.env`, the cancellation pre-check — see [Agent System & MCP Connectors](../concepts/agent-system.md). For the per-agent API-key table and the validation gate, see [Environment Variables & Secrets](../operations/environment-variables.md) under *Per-agent API keys*. For the connectors schema that each wrapper receives as `mcpServers`, see [Agent System & MCP Connectors → From `Connector` to per-CLI MCP config](../concepts/agent-system.md#from-connector-to-per-cli-mcp-config).

## The shared contract

All six wrappers export an `executeXxxInSandbox` function with the same positional signature and return a `Promise` of `AgentExecutionResult`. The dispatcher's `switch` block in `lib/sandbox/agents/index.ts` calls one of them per agent type.

The return shape is `AgentExecutionResult`, defined in [`lib/sandbox/types.ts`](repo://lib/sandbox/types.ts#L43-L53):

```ts
type AgentExecutionResult = {
  success: boolean
  output?: string
  agentResponse?: string
  cliName?: string
  changesDetected?: boolean
  error?: string
  streamingLogs?: unknown[]
  logs?: LogEntry[]
  sessionId?: string
}
```

The dispatcher threads the parameters through; not every wrapper consumes all of them.

`taskId` and `agentMessageId` are only used by the streaming wrappers (Claude, Cursor, Copilot) to update a single `taskMessages` row in place. `isResumed` and `sessionId` are forwarded by every wrapper except Gemini, which has no resume mechanism.

The wrappers agree on four invariants:

1. **Project-directory cwd.** Every command that touches repo files runs in `PROJECT_DIR = /vercel/sandbox/project` via the `runInProject` helper in `lib/sandbox/commands.ts`. Installation commands that must run before the repo is guaranteed to be present use `runCommandInSandbox` instead.
2. **Cancellation contract.** The wrappers never check `onCancellationCheck` themselves. The dispatcher does it once before the `switch` and returns a failed result.
3. **`changesDetected` is computed from git, not the CLI exit code.** Every wrapper ends with `git status --porcelain` and treats any non-empty output as "changes were made".
4. **`success` is the CLI's success, not git's.** A wrapper returns `success: true` whenever its CLI process exited 0, even when `changesDetected` is false — agents often answer a question without editing files.

Three additional patterns apply to every wrapper:

- **Already-installed check.** Every wrapper starts with a CLI install check (or a `sh -c 'which ...'` variant). If the binary is present, installation is skipped — this is how keep-alive sandboxes survive follow-up turns without re-downloading the CLI.
- **MCP is opt-in.** The `if (mcpServers && mcpServers.length > 0)` guard around the MCP config block is the same in all six wrappers. Empty arrays skip config-file generation entirely.
- **Secret redaction.** Every command line and every stdout/stderr line that the wrapper echoes into the logger passes through `redactSensitiveInfo` (`lib/utils/logging.ts`). The wrappers also build a redacted version of the command string before `logger.command(...)` so the persisted log never carries an API key in the clear.

## Per-agent capability matrix

The six wrappers differ along five dimensions: how they install, what credential they require, whether they stream into the chat (`taskMessages`), what flag they use to resume, and where their MCP config lives. The table below summarizes what each one does; the sections that follow explain the non-obvious mechanics. The single-column source files referenced below are [`lib/sandbox/agents/claude.ts`](repo://lib/sandbox/agents/claude.ts), [`codex.ts`](repo://lib/sandbox/agents/codex.ts), [`copilot.ts`](repo://lib/sandbox/agents/copilot.ts), [`cursor.ts`](repo://lib/sandbox/agents/cursor.ts), [`gemini.ts`](repo://lib/sandbox/agents/gemini.ts), and [`opencode.ts`](repo://lib/sandbox/agents/opencode.ts).

| Agent | Install | Required env var(s) | Streams to `taskMessages` | Resumes via | MCP config path |
| --- | --- | --- | --- | --- | --- |
| `claude` | `curl -fsSL https://claude.ai/install.sh \| bash` | `AI_GATEWAY_API_KEY` | Yes (stream-json verbose, detached) | `--resume SESSION_ID` | `claude mcp add` (CLI command) |
| `codex` | `npm install -g @openai/codex` | `AI_GATEWAY_API_KEY` (must start with `sk-` or `vck_`) | No (synchronous `runInProject`) | `codex resume --last` (ignores `sessionId`) | `~/.codex/config.toml` (TOML) |
| `copilot` | `npm install -g @github/copilot` | `GH_TOKEN` or `GITHUB_TOKEN` | Yes (text-based, `--no-color`) | `--resume SESSION_ID` | `~/.copilot/mcp-config.json` via `--additional-mcp-config @` |
| `cursor` | `curl https://cursor.com/install -fsS \| bash -s -- --verbose` | `CURSOR_API_KEY` | Yes (`--output-format stream-json`, detached) | `--resume SESSION_ID` | `~/.cursor/mcp.json` |
| `gemini` | `npm install -g @google/gemini-cli` | One of `GEMINI_API_KEY`, `GOOGLE_API_KEY` + `GOOGLE_GENAI_USE_VERTEXAI`, `GOOGLE_CLOUD_PROJECT` | No (synchronous `runCommandInSandbox`) | None | `~/.gemini/settings.json` |
| `opencode` | `npm install -g opencode-ai` | `OPENAI_API_KEY` *or* `ANTHROPIC_API_KEY` | No (synchronous `runCommandInSandbox`) | `--session SESSION_ID` or `--continue` | `~/.opencode/config.json` |

Of the six wrappers, three stream into the chat (`claude`, `cursor`, `copilot`) and three do not (`codex`, `gemini`, `opencode`). All six except Gemini support some form of session resumption, and only Codex silently rewrites an explicit `sessionId` to "the most recent session".

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    D["dispatcher<br/>executeAgentInSandbox"]
    C["claude.ts<br/>stream-json<br/>--resume"]
    X["codex.ts<br/>sync<br/>codex resume --last"]
    P["copilot.ts<br/>--no-color<br/>--resume"]
    U["cursor.ts<br/>stream-json<br/>--resume"]
    G["gemini.ts<br/>sync<br/>no resume"]
    O["opencode.ts<br/>sync<br/>--session"]
    DB[("taskMessages<br/>row updates")]
    R[("git status<br/>after run")]
    D --> C & X & P & U & G & O
    C & P & U -. in-place .-> DB
    C & X & P & U & G & O --> R
```

*Diagram: the dispatcher's switch routes one wrapper per agent type. Only Claude, Cursor, and Copilot write incrementally into a `taskMessages` row; every wrapper finishes with `git status --porcelain` to compute `changesDetected`.*

## Claude

Claude is the most fully featured wrapper and the only one that owns its install path through a dedicated exported helper (`installClaudeCLI`).

### Installation and authentication

The wrapper installs Claude via the official `https://claude.ai/install.sh` script ([`claude.ts:80`](repo://lib/sandbox/agents/claude.ts#L80)) and writes a config file at `$HOME/.config/claude/config.json` ([`claude.ts:151-L158`](repo://lib/sandbox/agents/claude.ts#L151-L158)). The config is a triple of `api_key`, `api_base_url`, and `default_model`:

```json
{
  "api_key": "<AI_GATEWAY_API_KEY value>",
  "api_base_url": "https://ai-gateway.vercel.sh",
  "default_model": "<selectedModel or 'claude-sonnet-4-5'>"
}
```

The wrapper additionally prefixes every shell invocation with `ANTHROPIC_API_KEY` and `ANTHROPIC_BASE_URL=https://ai-gateway.vercel.sh`, so the Claude CLI sees the same gateway through two channels (config file *and* env). A `claude --version` smoke test runs after config-file creation to confirm auth ([`claude.ts:168-L176`](repo://lib/sandbox/agents/claude.ts#L168-L176)). The dispatcher's pre-flight `validateEnvironmentVariables` requires `AI_GATEWAY_API_KEY` for this agent; the wrapper additionally fails fast inside `executeClaudeInSandbox` if the key is missing ([`claude.ts:241-L248`](repo://lib/sandbox/agents/claude.ts#L241-L248)).

### MCP wiring

MCP servers are registered with the Claude CLI itself via `claude mcp add` rather than by writing a config file ([`claude.ts:99-L148`](repo://lib/sandbox/agents/claude.ts#L99-L148)). Local STDIO servers are appended with their env vars as `--env KEY=VALUE` flags; remote HTTP servers use `--transport http` and forward `oauthClientSecret` / `oauthClientId` as `--header "Authorization: Bearer …"` / `--header "X-Client-ID: …"`.

### Streaming and output parsing

The Claude wrapper runs the CLI with `detached: true`, `--output-format stream-json --verbose`, and a `Writable` capture stream attached to stdout ([`claude.ts:278-L423`](repo://lib/sandbox/agents/claude.ts#L278-L423)). Two event shapes are handled:

- **`type: 'assistant'`** — `message.content[].type === 'text'` blocks accumulate into `accumulatedContent`; `type === 'tool_use'` blocks emit a human-readable status line (`Editing PATH`, `Reading PATH`, `Running: COMMAND`, `Grep: PATTERN`, etc.).
- **`type: 'result'`** — the wrapper captures `parsed.session_id` into `extractedSessionId` and flips `isCompleted = true`. A 1-second polling loop exits the moment `isCompleted` flips.

When streaming is active the returned `AgentExecutionResult` sets `agentResponse: undefined` to avoid duplicating content already in chat. The `sessionId` field is always populated on success.

### Resume

When `isResumed` is true, the wrapper appends `--resume SESSION_ID` (quoted) if `sessionId` is present, or bare `--resume` to the command ([`claude.ts:280-L293`](repo://lib/sandbox/agents/claude.ts#L280-L293)). A follow-up turn without an explicit id therefore continues the most recent Claude conversation.

## Codex

Codex is the only wrapper that branches on the *shape* of the API key and the only one that writes a TOML config file. It is also the only wrapper that ignores an explicit `sessionId` and uses a "resume the last conversation" flag instead.

### Installation and authentication

The wrapper installs `@openai/codex` globally via npm ([`codex.ts:52`](repo://lib/sandbox/agents/codex.ts#L52)). It then refuses to proceed if `AI_GATEWAY_API_KEY` is missing or if the key does not start with `sk-` (OpenAI) or `vck_` (Vercel AI Gateway) — the two prefixes the wrapper recognizes ([`codex.ts:88-L105`](repo://lib/sandbox/agents/codex.ts#L88-L105)).

The config file at `~/.codex/config.toml` is generated as a string template that branches on the key prefix:

- **`vck_` (Vercel AI Gateway)** — `model_provider = "vercel-ai-gateway"`, `base_url = "https://ai-gateway.vercel.sh/v1"`, `env_key = "AI_GATEWAY_API_KEY"`, `wire_api = "chat"`.
- **`sk-` (OpenAI)** — `model_provider = "openai"`, `base_url = "https://api.openai.com/v1"`, `env_key = "AI_GATEWAY_API_KEY"`, `wire_api = "responses"`.

Both branches emit a `[debug]` section with `log_requests = true` for diagnostics. `selectedModel` defaults to `openai/gpt-4o` if not provided ([`codex.ts:150-L183`](repo://lib/sandbox/agents/codex.ts#L150-L183)). The final `codex exec` invocation runs under a shell that prefixes `AI_GATEWAY_API_KEY`, `HOME=/home/vercel-sandbox`, and `CI=true` ([`codex.ts:307-L308`](repo://lib/sandbox/agents/codex.ts#L307-L308)).

### MCP wiring

MCP servers are emitted as `[mcp_servers.SERVER_NAME]` blocks in the same `config.toml` ([`codex.ts:186-L235`](repo://lib/sandbox/agents/codex.ts#L186-L235)). If any server is remote, the wrapper prepends `experimental_use_rmcp_client = true` to enable Codex's remote MCP client. Local servers set `command`, `args`, and `env`; remote servers set `url` and `bearer_token`.

### Streaming and output parsing

Codex is not streaming. The wrapper invokes `runInProject` synchronously and accumulates the full stdout into `result.output` ([`codex.ts:311`](repo://lib/sandbox/agents/codex.ts#L311)). The `sessionId` is recovered with a regex on the output text — `(?:session[_\s-]?id|Session)[:\s]+([a-f0-9-]+)` ([`codex.ts:336-L343`](repo://lib/sandbox/agents/codex.ts#L336-L343)).

### Resume

When `isResumed` is true, the wrapper switches from `codex exec --dangerously-bypass-approvals-and-sandbox` to `codex resume --last` ([`codex.ts:284-L292`](repo://lib/sandbox/agents/codex.ts#L284-L292)). The explicit `sessionId` argument is *not* passed — Codex's resume subcommand either picks from a picker or uses `--last`. This means a follow-up turn always continues the most recent Codex session regardless of which session the original run belonged to.

## Copilot

The Copilot wrapper authenticates via the user's GitHub OAuth token and is the only one whose token source is fetched dynamically by the dispatcher (the `getUserGitHubToken` import in [`lib/sandbox/agents/index.ts:51-L54`](repo://lib/sandbox/agents/index.ts#L51-L54)).

### Installation and authentication

The wrapper installs `@github/copilot` via npm ([`copilot.ts:62`](repo://lib/sandbox/agents/copilot.ts#L62)) and refuses to proceed without `GH_TOKEN` or `GITHUB_TOKEN` ([`copilot.ts:93-L100`](repo://lib/sandbox/agents/copilot.ts#L93-L100)). Both env vars are forwarded to the copilot process (the CLI accepts either).

### MCP wiring

Copilot is the only wrapper that passes its MCP config via a CLI flag rather than relying on a well-known path. The wrapper writes `~/.copilot/mcp-config.json` and passes `--additional-mcp-config @PATH` to the CLI ([`copilot.ts:171-L184`](repo://lib/sandbox/agents/copilot.ts#L171-L184)). The JSON shape is:

- Local: `type: 'stdio'`, with `command`, `args`, `env`, and `tools: []` (empty `tools` means expose all).
- Remote: `type: 'http'`, with `url`, optional `headers`, and `tools: []`.

### Streaming and output parsing

Copilot streams text (not JSON) via `--no-color`, and the wrapper filters out diff-box lines (those containing box-drawing characters like `╭╰│─═`) before appending to `accumulatedContent` ([`copilot.ts:204-L246`](repo://lib/sandbox/agents/copilot.ts#L204-L246)). Each non-diff line is appended verbatim; lines starting with `●` or `✓` (thought/action markers) get a blank line prepended for readability. The accumulated content is wrapped in a `<pre class="whitespace-pre-wrap font-sans text-xs">` tag in the taskMessages row so the chat UI renders it monospace.

### Resume

When `isResumed` is true, the wrapper appends `--resume SESSION_ID` to the args ([`copilot.ts:287-L289`](repo://lib/sandbox/agents/copilot.ts#L287-L289)).

## Cursor

The Cursor wrapper installs into a non-standard path (`~/.local/bin/cursor-agent`) and runs the CLI by absolute path rather than relying on `PATH`.

### Installation and authentication

The wrapper runs `curl https://cursor.com/install -fsS | bash -s -- --verbose` and then performs a sequence of post-install checks (`ls -la ~/.local/bin/`, `which cursor-agent` with `PATH` extended) to confirm the binary is reachable ([`cursor.ts:85-L106`](repo://lib/sandbox/agents/cursor.ts#L85-L106)). If `which` fails, the wrapper searches `/usr/local/bin`, `/home`, `/opt`, and `~/.local/bin` before giving up.

Once available, the wrapper invokes `/home/vercel-sandbox/.local/bin/cursor-agent` directly (no `sh -c` wrapper) with `CURSOR_API_KEY` in the env ([`cursor.ts:461-L472`](repo://lib/sandbox/agents/cursor.ts#L461-L472)). `CURSOR_API_KEY` is the only credential this wrapper accepts.

### MCP wiring

MCP servers are written to `~/.cursor/mcp.json`. The shape is the simplest of the six — local servers use `{ command, args, env }`; remote servers use `{ url, headers? }` ([`cursor.ts:184-L239`](repo://lib/sandbox/agents/cursor.ts#L184-L239)). There is no `type` discriminator.

### Streaming and output parsing

Cursor runs with `detached: true`, `-p`, `--force`, and `--output-format stream-json`. The wrapper parses three event types ([`cursor.ts:324-L428`](repo://lib/sandbox/agents/cursor.ts#L324-L428)):

- **`type: 'tool_call', subtype: 'started'`** — generates a status message based on the tool name (`editToolCall` becomes "Editing PATH", `readToolCall` becomes "Reading PATH", `runCommandToolCall` becomes "Running command", `shellToolCall` becomes "Running: COMMAND", `grepToolCall` becomes "Searching for: PATTERN", `semSearchToolCall` becomes "Searching codebase: QUERY", `globToolCall` becomes "Finding files: PATTERN", and other tool calls are normalised by stripping the `ToolCall` suffix).
- **`type: 'assistant', message.content[]`** — extracts text blocks and appends them to `accumulatedContent`.
- **`type: 'result'`** — captures `session_id` into `extractedSessionId` and flips `isCompleted` when the result's `subtype` is `success` or `is_error: false`.

The wrapper also hard-stops the polling loop after 240 seconds (4 minutes) to leave headroom before the 5-minute sandbox timeout ([`cursor.ts:486-L493`](repo://lib/sandbox/agents/cursor.ts#L486-L493)).

### Resume

When `isResumed` is true, the wrapper appends `--resume SESSION_ID` to the args ([`cursor.ts:456-L458`](repo://lib/sandbox/agents/cursor.ts#L456-L458)).

## Gemini

Gemini is the most permissive wrapper about authentication and the only one that does not implement resumption.

### Installation

The wrapper installs `@google/gemini-cli` via npm on first use ([`gemini.ts:64`](repo://lib/sandbox/agents/gemini.ts#L64)). The CLI is invoked synchronously with `runCommandInSandbox` (not `runInProject`) because the wrapper does not need the project cwd.

### Authentication, in priority order

Gemini's auth chain accepts any of three credential shapes ([`gemini.ts:178-L200`](repo://lib/sandbox/agents/gemini.ts#L178-L200)):

1. **`GEMINI_API_KEY`** — direct Gemini API. The wrapper logs "Using Gemini API key authentication" and exports the key as `GEMINI_API_KEY`.
2. **`GOOGLE_API_KEY` + `GOOGLE_GENAI_USE_VERTEXAI=true`** — Vertex AI path. Exports both env vars.
3. **`GOOGLE_CLOUD_PROJECT`** — OAuth / Code Assist path. Exports the project id only; the user must complete an interactive login on first use.
4. **(default)** — no env vars; the wrapper logs that it will attempt OAuth and lets the CLI prompt.

This makes Gemini the only wrapper whose `validateEnvironmentVariables` check is more permissive than the wrapper itself — the env gate only enforces `GEMINI_API_KEY`, while the wrapper also accepts the Vertex AI and OAuth paths.

### MCP wiring

MCP servers go to `~/.gemini/settings.json`. Local servers use `{ command, args, env }`; remote servers use `{ httpUrl, headers? }` ([`gemini.ts:94-L148`](repo://lib/sandbox/agents/gemini.ts#L94-L148)). The remote `httpUrl` key name is unique to Gemini.

### Streaming and output parsing

Gemini is not streaming. The wrapper runs the CLI synchronously and reads stdout/stderr from the `CommandResult`.

### The fallback chain

If the first invocation fails with a "Tool not found in registry" error (a known sandbox-specific issue), the wrapper retries twice: first with `--approval-mode auto_edit -o text`, then with the minimal flag set ([`gemini.ts:240-L264`](repo://lib/sandbox/agents/gemini.ts#L240-L264)). The successful invocation's output is returned verbatim as `agentResponse`.

### No resume mechanism

The Gemini wrapper does not accept `isResumed`, `sessionId`, or `taskId`. Follow-up turns that use Gemini rely on the prompt being prefixed with the last five messages of the conversation, which the continue route assembles before calling `executeAgentInSandbox` ([`continue/route.ts:248-L284`](repo://app/api/tasks/[taskId]/continue/route.ts#L248-L284)).

## OpenCode

The OpenCode wrapper is the only one that registers credentials via the OpenCode CLI's own `auth add` command and the only one whose required-key gate is "either-or" between two different providers.

### Installation and authentication

The wrapper installs `opencode-ai` via npm ([`opencode.ts:88`](repo://lib/sandbox/agents/opencode.ts#L88)) and refuses to run if *neither* `OPENAI_API_KEY` nor `ANTHROPIC_API_KEY` is set ([`opencode.ts:60-L69`](repo://lib/sandbox/agents/opencode.ts#L60-L69)). After install, for each key that is present it pipes the key into `opencode auth add PROVIDER` so the OpenCode CLI stores it as its own credential ([`opencode.ts:248-L283`](repo://lib/sandbox/agents/opencode.ts#L248-L283)). The wrapper also exports `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` on the final shell invocation so the `opencode run` command picks them up directly without reading the store.

### Streaming and output parsing

OpenCode is not streaming. The wrapper invokes `opencode run --model MODEL` synchronously via `runCommandInSandbox` and reads the full stdout/stderr from the result. The `sessionId` is recovered with the same regex as Codex — `(?:session[_\s-]?id|Session)[:\s]+([a-f0-9-]+)` ([`opencode.ts:380-L386`](repo://lib/sandbox/agents/opencode.ts#L380-L386)). `selectedModel` is forwarded verbatim — OpenCode accepts provider/model strings like `anthropic/claude-sonnet-4-5` or `openai/gpt-4o`.

### MCP wiring

MCP servers are written to `~/.opencode/config.json`. Unlike the other wrappers, OpenCode uses a typed `type: 'local' | 'remote'` discriminator and a single `command: string[]` (executable and args bundled) ([`opencode.ts:154-L213`](repo://lib/sandbox/agents/opencode.ts#L154-L213)). Local servers add `environment?`; remote servers add `url` and `headers?`.

### Resume

When `isResumed && sessionId`, the wrapper appends `--session SESSION_ID`. When `isResumed` without `sessionId`, it appends bare `--continue`, which tells OpenCode to continue the most recent session for the working directory ([`opencode.ts:331-L344`](repo://lib/sandbox/agents/opencode.ts#L331-L344)).

## Common failure and fallback behavior

Three failure modes are handled uniformly across the wrappers:

- **Missing required env var.** The wrapper returns `{ success: false, error: '<env var name> environment variable is required but not found', cliName, changesDetected: false }`. The error message names the missing variable so the user can act on it without consulting the source.
- **Install failure.** Same shape; the error message names the npm/curl failure from the underlying installer.
- **CLI not found after install.** Same shape; a distinct error message prevents the "install reported success but the binary is missing" class of bug from being mistaken for an auth error.

Two additional behaviors are uniform:

- **Successful CLI run with `changesDetected: false`.** This is **not** an error. The wrapper returns `success: true` and the message body annotates `(No changes made)`. A common false-positive to avoid is treating a question-answering agent run as a failure just because no files changed.
- **Logged but unobservable failures.** Per-CLI config-file writes (e.g. `~/.cursor/mcp.json`) log success/failure to the task logger but never short-circuit the wrapper. A missing MCP config simply means the agent runs without MCP tools, and the user-facing error from the CLI is what surfaces.

## Extension points

Adding a new agent CLI requires the same three-file pattern as adding a new dispatcher route. See [Agent System & MCP Connectors → Extension points](../concepts/agent-system.md#extension-points) for the full procedure. The per-CLI surface that any new wrapper must conform to:

1. Accept the standard parameter list from the dispatcher (`sandbox`, `instruction`, `logger`, `selectedModel`, `mcpServers`, `isResumed`, `sessionId`, `taskId`, `agentMessageId`).
2. Implement the install check pattern (binary already present) so keep-alive sandboxes do not re-download.
3. Write the MCP config in whatever dialect the new CLI speaks, behind an `if (mcpServers && mcpServers.length > 0)` guard.
4. Decide between synchronous execution and streaming (`detached: true` plus a `Writable` stream). Streaming is only worth wiring if the CLI exposes a line-oriented JSON or text stream; otherwise the synchronous path is correct.
5. Compute `changesDetected` from `git status --porcelain` after the run.
6. Extract `sessionId` if the CLI supports resumption; otherwise the wrapper ignores `isResumed` / `sessionId` like Gemini does.
7. Add a row to the `validateEnvironmentVariables` agent-key map in `lib/sandbox/config.ts` so a missing key surfaces at sandbox-creation time rather than at agent execution time.

All secrets that reach the wrapper must remain in the wrapper — never paste decrypted credentials into the wiki, the logger, or the persisted `taskMessages` row.
