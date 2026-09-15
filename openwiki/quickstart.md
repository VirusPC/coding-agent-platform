---
type: "Reference"
title: "OpenWiki Quickstart"
description: "One-paragraph orientation to the coding-agent-template repository, a five-step reading tour, and a routing map into the seven wiki domains (architecture, concepts, integrations, operations, systems, testing, workflows)."
tags: [quickstart, navigation, route-map, overview, index]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-8037e2358a2c4f9b2c722a11
    resource: repo://AGENTS.md
  - id: openwiki-source-c7baae88c7e2880cc027c016
    resource: repo://app/api/tasks/route.ts
  - id: openwiki-source-f4a535c4711843f9246eaee1
    resource: repo://lib/auth/providers.ts
  - id: openwiki-source-c5e751251b730ee67d7f9fb1
    resource: repo://lib/crypto.ts
  - id: openwiki-source-2972832ca9085b9e225f8b49
    resource: repo://lib/jwe/encrypt.ts
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
  - id: openwiki-source-4c0dfa7b4928caf4818e73f8
    resource: repo://lib/utils/logging.ts
  - id: openwiki-source-5b54a58d1b51cd490b0e7162
    resource: repo://package.json
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# OpenWiki Quickstart

The **Coding Agent Template** (see [`package.json`](repo://package.json), [`README.md`](repo://README.md)) is a Next.js 15 / React 19 application that turns a natural-language prompt into a real code change on a Git branch: a user signs in with GitHub or Vercel OAuth, picks a repository and an agent CLI (Claude Code, Codex, Copilot, Cursor, Gemini, or opencode), and a background worker spins up a remote **Vercel Sandbox**, runs the chosen agent against the cloned repository, and pushes the resulting commits to a new branch. Authentication is mandatory, every per-user credential is encrypted at rest with AES-256-CBC under [`ENCRYPTION_KEY`](repo://lib/crypto.ts), and the session cookie is a JWE signed with [`JWE_SECRET`](repo://lib/jwe/encrypt.ts). The single architectural fact that drives everything else is that **the HTTP response returns before the work begins**: the route handler inserts a `tasks` row (`status='pending'`), hands the heavy work to `next/server`'s `after()` callbacks, and the worker mutates that same row all the way to `completed` / `error` / `stopped`. This wiki exists so a coding agent or a new contributor can navigate that pipeline quickly without rereading the whole codebase.

## How to read this wiki in five steps

1. **Start with the system shape.** Read [System Overview](./architecture/system-overview.md) for the one-screen mental model: the request path (client → route handler → `after()` worker → sandbox → agent CLI → Git push) and the data path (the `tasks` row is the contract between the synchronous API and the asynchronous worker).
2. **Trace one task end to end.** Open [Runtime Flow: Request → Sandbox → Agent → Push](./architecture/runtime-flow.md) for the phased walkthrough, including every `isTaskStopped` checkpoint and what each `after()` callback owns.
3. **Anchor on the task row.** Skim [Task Lifecycle & Status Model](./concepts/tasks-lifecycle.md) for the five statuses (`pending` → `processing` → `completed` | `error` | `stopped`), the append-only `logs` JSONB column, the `deletedAt` soft-delete tombstone, and the soft-delete vs. `status='stopped'` distinction.
4. **Walk a happy path through the workflow.** Follow [Create and Run a Task](./workflows/create-and-run-task.md) to see how a form submission becomes a pushed branch: `POST /api/tasks` → three `after()` callbacks (branch name, title, worker) → `createSandbox` → `executeAgentInSandbox` → `pushChangesToBranch` → client polling.
5. **Follow the branch.** Once a task has a pushed branch, [Branch → Pull Request → Review → Merge/Close](./workflows/open-pull-request.md) covers the `app/api/tasks/[taskId]/pr`, `/merge-pr`, `/close-pr`, and `/sync-pr` routes. For follow-up messages and the `keepAlive` flag's two effects on the sandbox, see [Follow-up Messages and Keep-Alive](./workflows/follow-up-and-keep-alive.md).

## Route map

The wiki is organized into seven domains. Start with the page that matches the question you are trying to answer; the most-cited pages from each domain are listed below.

### Architecture

Mental-model pages for the whole system.

- [System Overview](./architecture/system-overview.md) — one-screen view of the request and data paths, plus a subsystem table.
- [Runtime Flow: Request → Sandbox → Agent → Push](./architecture/runtime-flow.md) — phased walkthrough of a single task, with every cancellation checkpoint.

### Concepts

Cross-cutting mental models that several subsystems share.

- [Task Lifecycle & Status Model](./concepts/tasks-lifecycle.md) — the `tasks` row as the unit of work.
- [Sandbox Lifecycle](./concepts/sandbox-lifecycle.md) — `createSandbox`, the in-memory `sandbox-registry`, `Sandbox.get()` reconnect, and shutdown.
- [Agent System & MCP Connectors](./concepts/agent-system.md) — the dispatcher in [`lib/sandbox/agents/index.ts`](repo://lib/sandbox/agents/index.ts) and how MCP servers are decrypted and injected.
- [Authentication & Sessions](./concepts/auth-and-sessions.md) — Vercel and GitHub OAuth, the JWE-encrypted session cookie, and the linking model.
- [Encryption & Log Redaction](./concepts/encryption-and-redaction.md) — AES-256-CBC at rest, JWE A256GCM for the session, and the `redactSensitiveInfo` scrubber.

### Integrations

External services the app depends on.

- [GitHub OAuth](./integrations/github-oauth.md) — app registration, scopes, and the connect flow.
- [Vercel OAuth](./integrations/vercel-oauth.md) — PKCE handshake via `arctic.OAuth2Client`.
- [Vercel AI Gateway](./integrations/vercel-ai-gateway.md) — branch name, title, and commit-message generation.
- [Vercel Sandbox](./integrations/vercel-sandbox.md) — the `@vercel/sandbox` SDK surface used by the worker.

### Operations

Runtime configuration, migrations, and policy.

- [Environment Variables & Secrets](./operations/environment-variables.md) — every env var, who reads it, and whether it is client-safe.
- [Database Migrations](./operations/database-migrations.md) — Drizzle generate / push / migrate and `scripts/migrate-production.ts`.
- [Security Policy & Log Redaction Rules](./operations/security-and-redaction.md) — the `AGENTS.md` rules in canonical form.

### Systems

Per-subsystem reference.

- [Database Schema (Drizzle + Postgres)](./systems/database-schema.md) — every table in [`lib/db/schema.ts`](repo://lib/db/schema.ts).
- [Task API Surface](./systems/task-api.md) — every route under `/api/tasks`.
- [Auth API Surface](./systems/auth-flow.md) — signin / callback / info / rate-limit / signout routes.
- [Sandbox Orchestration](./systems/sandbox-orchestration.md) — `createSandbox`, `commands.ts`, package-manager and port detection.
- [Agent Implementations](./systems/agent-implementations.md) — the six CLIs under `lib/sandbox/agents/`.
- [Connectors (MCP Servers)](./systems/connectors-mcp.md) — the `connectors` table and server actions.
- [API Keys Management](./systems/api-keys-management.md) — the `keys` table and `/api/api-keys`.
- [GitHub Integration (Octokit & PR Workflow)](./systems/github-integration.md) — Octokit helpers and the repo browser.
- [Live Editor Surface](./systems/live-editor-surface.md) — file browser, Monaco editor, diff viewer, terminal, LSP bridge.
- [AI Content Generation](./systems/ai-content-generation.md) — the three `lib/utils/` generators and their fallbacks.
- [Client State (Jotai, Cookies, Hooks)](./systems/client-state.md) — atoms, the cookies utility, and the `useTask` polling hook.

### Testing

- [Testing Strategy & Coverage](./testing/strategy-and-coverage.md) — current posture (no formal unit/integration tests; `pnpm type-check`, `lint`, `format`, and `build` are the gates) and where to add tests per subsystem.

### Workflows

End-to-end procedures that span multiple subsystems.

- [Create and Run a Task](./workflows/create-and-run-task.md) — the happy path.
- [Branch → Pull Request → Review → Merge/Close](./workflows/open-pull-request.md) — the PR lifecycle.
- [Follow-up Messages and Keep-Alive](./workflows/follow-up-and-keep-alive.md) — `/continue`, `keepAlive`, and the stop action.
- [Add an MCP Server](./workflows/add-mcp-server.md) — connector configuration and injection into the agent CLI.
- [Connect GitHub to a Vercel-Signed-In User](./workflows/connect-github.md) — the secondary GitHub-connect flow.

## Rules every contributor must follow

These come from [`AGENTS.md`](repo://AGENTS.md) and are non-negotiable for any agent that edits the source tree or logs:

- **No dynamic values in logs.** Every `logger.*`, `console.*`, and `logger.updateProgress` call must use a static string — never a template literal with `${}`. Logs are returned to the UI, so dynamic values can leak user IDs, file paths, branch names, or credentials. `redactSensitiveInfo` in [`lib/utils/logging.ts`](repo://lib/utils/logging.ts) is a backup; static strings are the primary defense.
- **Never run dev servers.** Do not invoke `pnpm dev`, `next dev`, `npm start`, or any long-running process. Use `pnpm build` to verify the production build, `pnpm type-check` for types, `pnpm lint` for lint, and let the user run the dev server themselves.
- **Run `pnpm format` after every TS/TSX edit.** The project uses Prettier (see `prettier` block in [`package.json`](repo://package.json)).
- **Keep secrets out of the client bundle.** Only `NEXT_PUBLIC_AUTH_PROVIDERS` and `NEXT_PUBLIC_GITHUB_CLIENT_ID` (or `NEXT_PUBLIC_VERCEL_CLIENT_ID`) may be prefixed with `NEXT_PUBLIC_`. Every other secret — `SANDBOX_VERCEL_TOKEN`, `JWE_SECRET`, `ENCRYPTION_KEY`, `POSTGRES_URL`, per-user API keys — is server-only.
- **Add UI components via the shadcn CLI.** Run `pnpm dlx shadcn@latest add <component-name>` rather than hand-rolling a wrapper for an existing primitive.
