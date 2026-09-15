---
type: concept
title: Agent System & MCP Connectors
description: How the dispatcher in lib/sandbox/agents/index.ts routes work to the per-CLI agent wrappers, how user MCP connectors are decrypted and threaded into each invocation, and how the temp-env-var pattern snapshots and restores process.env around agent execution.
tags: [agents, mcp, connectors, sandbox, dispatcher, encryption]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-3401b854c53347910d2c84ce
    resource: repo://app/api/tasks/%5BtaskId%5D/continue/route.ts
  - id: openwiki-source-c7baae88c7e2880cc027c016
    resource: repo://app/api/tasks/route.ts
  - id: openwiki-source-176adf04be215509357457c9
    resource: repo://lib/actions/connectors.ts
  - id: openwiki-source-526feb2334febba699169a23
    resource: repo://lib/db/schema.ts
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
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Agent System & MCP Connectors

This page describes the single dispatcher that decides which coding agent CLI a task runs, the `AgentType` union it switches on, the shape of the `Connector` rows that represent user-owned MCP servers, where their secret fields are decrypted, and the temp-env-var pattern that snapshots and restores `process.env` around each agent execution.

For the step-by-step walkthrough of where this fits in a task, see [Runtime Flow: Request → Sandbox → Agent → Push](../architecture/runtime-flow.md). For per-CLI details (auth requirements, output streaming, resumption) see [Agent Implementations](../systems/agent-implementations.md). For the broader MCP connector lifecycle, see [Connectors & MCP](../systems/connectors-mcp.md). For the encryption scheme itself, see [Encryption and Redaction](../concepts/encryption-and-redaction.md).

## The dispatcher

The dispatcher is `executeAgentInSandbox` in `lib/sandbox/agents/index.ts`. It is the only entry point the task worker uses to run an agent, and it is the single place that knows about the full set of supported CLIs.

Its signature is:

```ts
executeAgentInSandbox(
  sandbox: Sandbox,
  instruction: string,
  agentType: AgentType,
  logger: TaskLogger,
  selectedModel?: string,
  mcpServers?: Connector[],
  onCancellationCheck?: () => Promise<boolean>,
  apiKeys?: {
    OPENAI_API_KEY?: string
    GEMINI_API_KEY?: string
    CURSOR_API_KEY?: string
    ANTHROPIC_API_KEY?: string
    AI_GATEWAY_API_KEY?: string
  },
  isResumed?: boolean,
  sessionId?: string,
  taskId?: string,
  agentMessageId?: string,
): Promise<AgentExecutionResult>
```

The two callers are `POST /api/tasks` and `PATCH /api/tasks/:taskId/continue`. Both fetch the current user's connected MCP connectors, decrypt them on the server, and pass the resulting array as the `mcpServers` argument. See [Runtime Flow: Request → Sandbox → Agent → Push](../architecture/runtime-flow.md) for how this sits inside the task lifecycle.

### `AgentType` union

The dispatcher is the canonical declaration of `AgentType`:

```ts
export type AgentType = 'claude' | 'codex' | 'copilot' | 'cursor' | 'gemini' | 'opencode'
```

Every per-CLI wrapper is reached through a single `switch` on this union. Adding a new agent means adding the string to the union, adding a `case` to the switch, exporting an `execute<Type>InSandbox` from a new `lib/sandbox/agents/<name>.ts`, extending the `selected_agent` enum on the `tasks` schema, and adding a validation rule in `lib/sandbox/config.ts`. The default branch returns a `{ success: false, error: 'Unknown agent type: ...' }` so an unrecognized value surfaces a structured error rather than throwing.

### Cancellation checkpoint

Before any per-CLI work begins, the dispatcher calls the optional `onCancellationCheck` callback. If it resolves to `true`, the dispatcher logs `Task was cancelled before agent execution` and returns a failed `AgentExecutionResult` with `cliName: agentType` and `changesDetected: false`. This is the authoritative pre-agent stop signal; the task worker checks it before invoking the dispatcher, and the dispatcher's own check is a defense-in-depth pass inside the same call site.

### The temp-env-var pattern

The dispatcher's signature accepts an `apiKeys` object, but the per-CLI wrappers read `process.env.OPENAI_API_KEY`, `process.env.ANTHROPIC_API_KEY`, and friends directly. The dispatcher bridges the two by temporarily overriding `process.env` and always restoring it in a `finally` block. This is the temp-env-var pattern, and it is the reason a single sandbox process can run user-scoped work without leaking the per-user keys into any other concurrent work in the same Node process.

```ts
const originalEnv = {
  OPENAI_API_KEY: process.env.OPENAI_API_KEY,
  GEMINI_API_KEY: process.env.GEMINI_API_KEY,
  CURSOR_API_KEY: process.env.CURSOR_API_KEY,
  ANTHROPIC_API_KEY: process.env.ANTHROPIC_API_KEY,
  AI_GATEWAY_API_KEY: process.env.AI_GATEWAY_API_KEY,
  GH_TOKEN: process.env.GH_TOKEN,
  GITHUB_TOKEN: process.env.GITHUB_TOKEN,
}

if (apiKeys?.OPENAI_API_KEY) process.env.OPENAI_API_KEY = apiKeys.OPENAI_API_KEY
// ... same for each key

if (githubToken) {
  process.env.GH_TOKEN = githubToken
  process.env.GITHUB_TOKEN = githubToken
}

try {
  switch (agentType) { /* ... */ }
} finally {
  process.env.OPENAI_API_KEY = originalEnv.OPENAI_API_KEY
  process.env.GEMINI_API_KEY = originalEnv.GEMINI_API_KEY
  process.env.CURSOR_API_KEY = originalEnv.CURSOR_API_KEY
  process.env.ANTHROPIC_API_KEY = originalEnv.ANTHROPIC_API_KEY
  process.env.AI_GATEWAY_API_KEY = originalEnv.AI_GATEWAY_API_KEY
  process.env.GH_TOKEN = originalEnv.GH_TOKEN
  process.env.GITHUB_TOKEN = originalEnv.GITHUB_TOKEN
}
```

Three properties matter for safe change:

1. **Snapshot before override.** Every variable the dispatcher might touch is captured in `originalEnv` before any assignment, including values that are `undefined` at entry.
2. **`finally` always runs.** The restore block is in a `finally` clause so a thrown error from a per-CLI wrapper still restores the environment. The dispatcher does not currently re-throw — it returns `{ success: false, error: ... }` — but the `finally` covers any future change.
3. **GitHub token has its own branch.** When `agentType === 'copilot'`, the dispatcher dynamically imports `getUserGitHubToken` from `@/lib/github/user-token` and, if it returns a value, sets both `GH_TOKEN` and `GITHUB_TOKEN` (the Copilot CLI accepts either). The restore block writes both back, so a pre-existing system value for either variable is preserved.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart TD
    A["executeAgentInSandbox<br/>called by task worker"] --> B["onCancellationCheck?"]
    B -- cancelled --> X["return AgentExecutionResult success=false"]
    B -- continue --> C["Snapshot originalEnv<br/>of 7 keys"]
    C --> D["Override process.env<br/>from apiKeys + githubToken"]
    D --> E["switch (agentType)"]
    E -- claude --> F1["executeClaudeInSandbox"]
    E -- codex --> F2["executeCodexInSandbox"]
    E -- copilot --> F3["executeCopilotInSandbox<br/>also sets GH_TOKEN/GITHUB_TOKEN"]
    E -- cursor --> F4["executeCursorInSandbox"]
    E -- gemini --> F5["executeGeminiInSandbox"]
    E -- opencode --> F6["executeOpenCodeInSandbox"]
    E -- unknown --> X2["return AgentExecutionResult success=false<br/>error: Unknown agent type"]
    F1 & F2 & F3 & F4 & F5 & F6 --> G["finally: restore originalEnv"]
    X2 --> G
    X --> G
```

*Diagram: control flow inside `executeAgentInSandbox`. The `finally` block always restores the snapshot regardless of which branch ran.*

## MCP connectors: shape, storage, and decryption

Each MCP server a user attaches to a task is a row in the `connectors` table. The table stores only the structural fields; the secret fields (`oauthClientSecret`, `env`) are encrypted at rest using `lib/crypto.ts`'s `encrypt` and stored as opaque strings.

### Connector row shape

```ts
{
  id: string
  userId: string
  name: string
  description: string | null
  type: 'local' | 'remote'          // MCP transport
  baseUrl: string | null            // remote only
  oauthClientId: string | null
  oauthClientSecret: string | null  // encrypted; decrypted before agent run
  command: string | null            // local only; executable + args in one string
  env: string | null                // encrypted JSON string of Record<string,string>; decrypted before agent run
  status: 'connected' | 'disconnected'
  createdAt: Date
  updatedAt: Date
}
```

Two shapes coexist on the same table:

- **Local (STDIO).** `type === 'local'`. `command` holds the executable and any arguments in a single shell-style string (e.g. `npx -y @modelcontextprotocol/server-filesystem /tmp`). `env`, if present, is a JSON-encoded `Record<string, string>` of variables to inject into the spawned process. Used for tools that are runnable in the sandbox.
- **Remote (HTTP/SSE).** `type === 'remote'`. `baseUrl` is the upstream MCP endpoint. Optional `oauthClientId` and `oauthClientSecret` are mapped to HTTP request headers when the agent talks to the server.

### Where decryption happens

Decryption is **not** the dispatcher's job. The dispatcher only receives an already-decrypted array of `Connector` objects. Decryption lives in two adjacent places:

1. **In `lib/actions/connectors.ts`** (`getConnectors`), used by the connector-management UI. Every returned row has `oauthClientSecret` and `env` decrypted before serialization to the client. The UI also has write paths (`createConnector`, `updateConnector`) that re-encrypt on save.
2. **In the task worker** (`app/api/tasks/route.ts` and `app/api/tasks/[taskId]/continue/route.ts`), where the worker selects `connectors` rows where `userId = current user AND status = 'connected'`, then maps each row to a decrypted shape:

   ```ts
   const userConnectors = await db.select().from(connectors)
     .where(and(eq(connectors.userId, session.user.id), eq(connectors.status, 'connected')))

   mcpServers = userConnectors.map((connector) => ({
     ...connector,
     env: connector.env ? JSON.parse(decrypt(connector.env)) : null,
     oauthClientSecret: connector.oauthClientSecret ? decrypt(connector.oauthClientSecret) : null,
   }))
   ```

   The task worker also persists the **ids only** into `tasks.mcpServerIds` so the UI can later render which MCP servers were attached, without keeping the decrypted values around.

The agent implementations therefore receive a `Connector[]` whose `env` is a parsed `Record<string, string>` and whose `oauthClientSecret` is the plaintext string. Examples in this wiki never include real secret values; the encryption layer is described in [Encryption and Redaction](../concepts/encryption-and-redaction.md).

## From `Connector` to per-CLI MCP config

The dispatcher is deliberately thin: it does not transform the `Connector[]`. Each per-CLI wrapper decides how to expose the connector to its CLI, because each CLI speaks a different config dialect. The contract that all of them share:

1. Normalize the server name with `server.name.toLowerCase().replace(/[^a-z0-9]/g, '-')`.
2. If `server.type === 'local'`: split `server.command` on whitespace into executable + args, and forward `server.env` as process env.
3. If `server.type === 'remote'`: use `server.baseUrl`, and translate `oauthClientSecret`/`oauthClientId` into HTTP headers (`Authorization: Bearer ...` and `X-Client-ID: ...`).
4. Write the result to the CLI's expected path before invoking it, or pass the path as a CLI flag.

The shape each CLI receives differs:

| Agent | Local transport | Remote transport | Where the config is written |
| --- | --- | --- | --- |
| `claude` | `claude mcp add <name> -- <command>` with `--env KEY=VAL` flags | `claude mcp add --transport http <name> <baseUrl>` with `--header` flags | Persisted in Claude CLI's internal store |
| `codex` | `[mcp_servers.<name>] command = "..."` `args = [...]` `env = { ... }` in `~/.codex/config.toml` | `[mcp_servers.<name>] url = "..."` `bearer_token = "..."` in `~/.codex/config.toml` (with `experimental_use_rmcp_client = true` if any remote) | `~/.codex/config.toml` |
| `copilot` | `{ type: 'stdio', command, args, env }` entry in JSON | `{ type: 'http', url, headers }` entry in JSON | `~/.copilot/mcp-config.json`, passed via `--additional-mcp-config @<path>` |
| `cursor` | `{ command, args, env }` entry in JSON | `{ url, headers? }` entry in JSON | `~/.cursor/mcp.json` |
| `gemini` | `{ command, args, env }` entry in JSON | `{ httpUrl, headers? }` entry in JSON | `~/.gemini/settings.json` |
| `opencode` | `{ type: 'local', command: [...], enabled: true, environment? }` entry in JSON | `{ type: 'remote', url, enabled: true, headers? }` entry in JSON | `~/.opencode/config.json` |

The Codex wrapper is the only one that mutates its `config.toml` template string; the others always write a fresh JSON file. The OpenCode wrapper is the only one that uses a typed `type: 'local' | 'remote'` discriminator and a `command: string[]` (rather than separate `command` and `args`). The Copilot wrapper is the only one that passes the config via a CLI flag rather than relying on a well-known file path.

For the per-CLI details of how `oauthClientSecret`/`oauthClientId` are mapped to headers and how `env` is forwarded, see [Agent Implementations](../systems/agent-implementations.md).

## Failure and cancellation behavior

- **Unknown `agentType`.** Returns `{ success: false, error: 'Unknown agent type: <name>', cliName: <name>, changesDetected: false }`. The `finally` still restores `process.env`.
- **Cancellation pre-check.** Returns `{ success: false, error: 'Task was cancelled', cliName: agentType, changesDetected: false }`. No per-CLI work is started.
- **Per-CLI failure.** The wrapper returns its own `AgentExecutionResult`. The dispatcher's `finally` restores `process.env` and the returned error is propagated to the caller unchanged.
- **Decryption failures.** Decryption is the caller's responsibility. If the task worker's decrypt throws (e.g. because `ENCRYPTION_KEY` is missing or the row is malformed), the catch in the task route logs `Warning: Could not fetch MCP servers, continuing without them` and the dispatcher is called with `mcpServers = []`. No retry is attempted; the task continues without MCP tools.
- **Empty connector list.** `mcpServers = []` is a valid value. Each per-CLI wrapper guards its MCP-config block with `if (mcpServers && mcpServers.length > 0)`, so an empty array simply skips config-file generation.

## Extension points

Adding a new agent CLI requires touching three files and one database constraint:

1. Create `lib/sandbox/agents/<name>.ts` exporting `execute<Type>InSandbox(sandbox, instruction, logger, selectedModel?, mcpServers?, isResumed?, sessionId?, taskId?, agentMessageId?)`. The `Connector[]` argument is already decrypted, so the wrapper can use `server.command`, `server.env`, `server.baseUrl`, `server.oauthClientSecret`, and `server.oauthClientId` directly.
2. Add a `case '<name>':` to the `switch` in `executeAgentInSandbox` (`lib/sandbox/agents/index.ts`), extending the `AgentType` union in the same file.
3. Extend the `selected_agent` enum in the `tasks` schema (`lib/db/schema.ts`) and the validation rule in `lib/sandbox/config.ts` (which is what rejects a request early if the chosen agent's required API key is missing).

Adding a new MCP server shape (for example, a new transport) instead of a new agent requires touching the schema, the `createConnector`/`updateConnector` write paths in `lib/actions/connectors.ts`, and the read-side decryption in both `getConnectors` and the task worker — plus each per-CLI wrapper's `if (server.type === 'local') ... else ...` block.
