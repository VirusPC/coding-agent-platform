---
type: system
title: API Keys Management
description: Per-user encrypted provider API keys (anthropic, openai, cursor, gemini, aigateway) backed by the keys table, the GET/POST/DELETE surface at /api/api-keys, the readiness check at /api/api-keys/check, the system-env fallback in getUserApiKeys()/getUserApiKey(), and the API Keys dialog UI used to save or clear them.
tags: [api-keys, keys-table, encryption, user-keys, dispatcher, agents, ui, settings, env-fallback]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-c3b0b55b9aee3000394cb341
    resource: repo://app/api/api-keys/check/route.ts
  - id: openwiki-source-0ff9b499fc0f87ce88645e73
    resource: repo://app/api/api-keys/route.ts
  - id: openwiki-source-3401b854c53347910d2c84ce
    resource: repo://app/api/tasks/%5BtaskId%5D/continue/route.ts
  - id: openwiki-source-c7baae88c7e2880cc027c016
    resource: repo://app/api/tasks/route.ts
  - id: openwiki-source-27bab22e7ee7cb47e623de70
    resource: repo://components/api-keys-dialog.tsx
  - id: openwiki-source-819e40f1dfdc216e6306b923
    resource: repo://lib/api-keys/user-keys.ts
  - id: openwiki-source-1f28dc9acb836da434189f77
    resource: repo://lib/db/migrations/0010_concerned_exodus.sql
  - id: openwiki-source-526feb2334febba699169a23
    resource: repo://lib/db/schema.ts
  - id: openwiki-source-a02d819c42a405c6b6116e8f
    resource: repo://lib/sandbox/agents/claude.ts
  - id: openwiki-source-1f4c6b999a2fa110a4b14e5f
    resource: repo://lib/sandbox/agents/codex.ts
  - id: openwiki-source-461c38d111a1a74442ab8847
    resource: repo://lib/sandbox/agents/cursor.ts
  - id: openwiki-source-7563e3ff5929a46c9db6c26a
    resource: repo://lib/sandbox/agents/gemini.ts
  - id: openwiki-source-f111cb78e7ed9c79f1238982
    resource: repo://lib/sandbox/agents/index.ts
  - id: openwiki-source-bbbc9cdf967cf52b393b8240
    resource: repo://lib/sandbox/agents/opencode.ts
  - id: openwiki-source-78237118aac1cf9686d70d4d
    resource: repo://lib/sandbox/config.ts
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# API Keys Management

Every agent CLI that the platform can launch needs at least one provider credential (`AI_GATEWAY_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, or `CURSOR_API_KEY`). The platform runs in two modes at once: the system operator can configure these credentials globally via environment variables, and each signed-in user can override any of them with their own key, stored encrypted in the `keys` table and surfaced through the API Keys dialog. This page describes the storage, the HTTP surface, the resolution order that lets a user key win over a system key, and the UI that lets the user manage them.

For the AES-256-CBC primitive that encrypts `keys.value` at rest, see [Encryption & Log Redaction](../concepts/encryption-and-redaction.md). For the catalog of environment variables and the validateEnvironmentVariables gate, see [Environment Variables & Secrets](../operations/environment-variables.md). For the per-CLI install/auth/stream mechanics that consume the resolved key, see [Agent Implementations](./agent-implementations.md).

## Overview

Five provider credentials can be stored per user (`anthropic`, `openai`, `cursor`, `gemini`, `aigateway`). They are independent of the user's OAuth identity: a user signed in with Vercel who has never touched the API Keys dialog will use only the system env values; a user who has saved an `aigateway` row will use their own value for that one provider and fall back to the system env for the others. There is no global "use only my keys" toggle — each provider is resolved independently.

The components involved are:

- **`keys` table** in `lib/db/schema.ts` — one row per `(userId, provider)` pair, value encrypted with AES-256-CBC.
- **`getUserApiKeys()` / `getUserApiKey(provider)`** in `lib/api-keys/user-keys.ts` — the server-side readers that overlay decrypted user values on top of `process.env.*`.
- **`/api/api-keys`** route handlers in `app/api/api-keys/route.ts` — `GET` lists saved providers, `POST` upserts one, `DELETE` removes one.
- **`/api/api-keys/check`** route handler in `app/api/api-keys/check/route.ts` — `GET ?agent=…&model=…` returns whether *some* key is resolvable for the given agent and optionally model, used by the new-task UI.
- **`executeAgentInSandbox`** dispatcher in `lib/sandbox/agents/index.ts` — snapshots and restores `process.env` so per-agent CLIs read the user key transparently.
- **`ApiKeysDialog`** component in `components/api-keys-dialog.tsx` — the modal the user opens from the user dropdown to save or clear keys.
- **`SignOut`** component in `components/auth/sign-out.tsx` — the dropdown that mounts the dialog.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    U["User opens API Keys dialog"] -->|"POST /api/api-keys"| P["/api/api-keys POST"]
    P -->|"encrypt(plaintext)"| K[("keys table<br/>value=ciphertext")]
    P -->|"toast success"| U

    N["New task UI"] -->|"GET /api/api-keys/check?agent=…"| C["/api/api-keys/check"]
    C -->|"getUserApiKey(provider)"| G["getUserApiKeys/Key"]
    G -->|"read user row, decrypt"| K
    G -->|"fallback"| E["process.env.*"]
    C -->|"hasKey, provider, agentName"| N

    T["POST /api/tasks"] -->|"getUserApiKeys()"| G
    T -->|"apiKeys param"| D["executeAgentInSandbox"]
    D -->|"temp process.env.*"| AG["agent CLI<br/>(claude, codex, …)"]
```

*Diagram: the three call sites that consume the keys table — the dialog's `POST` writes rows, the readiness check answers "can this user run this agent?", and the task route fetches every key and passes them through the dispatcher so each CLI reads the right credential.*

## Schema: the `keys` table

The `keys` table is defined alongside the rest of the Drizzle schema in [`lib/db/schema.ts`](repo://lib/db/schema.ts#L318-L336) and is materialized in migration [`0010_concerned_exodus.sql`](repo://lib/db/migrations/0010_concerned_exodus.sql#L15-L22). Each row stores one user's credential for one provider:

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `text PRIMARY KEY` | Generated via `nanoid()` at insert time by the route handler. |
| `user_id` | `text NOT NULL` | FK to `users.id` with `ON DELETE CASCADE`. |
| `provider` | `text NOT NULL` | Enum restricted to `'anthropic' \| 'openai' \| 'cursor' \| 'gemini' \| 'aigateway'`. |
| `value` | `text NOT NULL` | The plaintext API key run through `lib/crypto.ts:encrypt` — never stored in the clear. |
| `created_at` | `timestamp DEFAULT now()` | Auto-stamped. |
| `updated_at` | `timestamp DEFAULT now()` | Bumped by `POST` whenever a row is updated. |

The unique index `keys_user_id_provider_idx` enforces the "one key per user per provider" invariant at the database level. This means a `POST` from the dialog can be implemented as a read-then-update-or-insert, but the upsert is straightforward because the route already checks for the existence of a matching row in JavaScript before deciding which branch to take.

The provider enum is intentionally lowercase (`anthropic`, `openai`, `cursor`, `gemini`, `aigateway`). The env-var names they map to are uppercase (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `CURSOR_API_KEY`, `GEMINI_API_KEY`, `AI_GATEWAY_API_KEY`) and the translation lives entirely inside `lib/api-keys/user-keys.ts`. Adding a sixth provider requires updating the enum in two places — the `pgEnum` declaration and the Zod schema — plus the switch in `getUserApiKeys` and the env-var map in both `getUserApiKeys` and `getUserApiKey`.

## HTTP surface: `/api/api-keys`

The route at [`app/api/api-keys/route.ts`](repo://app/api/api-keys/route.ts) exports three handlers. Every handler requires a signed-in session and reads the user from `getSessionFromReq(req)`; missing or invalid sessions return `401 Unauthorized`.

### `GET /api/api-keys`

Returns the set of providers the current user has saved, **without the key values**. The handler selects `(provider, createdAt)` from the `keys` table scoped to `session.user.id` and returns `{ success: true, apiKeys: [{ provider, createdAt }, …] }`. This is what the dialog uses to decide which rows to render as already-saved (the input becomes disabled and a "Clear" button replaces "Save").

The response deliberately omits `value`. There is no read path that decrypts a single user's key for the API surface; the key is only decrypted at the call sites that need it (the dispatcher or the readiness check), which are server-to-server and never expose the plaintext to the client.

### `POST /api/api-keys`

Body shape: `{ provider, apiKey }`. The handler:

1. Validates that `provider` is one of the five allowed strings and `apiKey` is a non-empty string; otherwise `400`.
2. Encrypts the plaintext with `encrypt(apiKey)` from `lib/crypto.ts`, producing the `<iv_hex>:<ciphertext_hex>` wire format.
3. Selects `keys` rows matching `(userId, provider)`; if one exists, updates `value` and `updatedAt`; otherwise inserts a new row with a fresh `nanoid()`.
4. Returns `{ success: true }` on the happy path, `500` on any unexpected error.

The upsert logic is implemented as a separate `SELECT` followed by `UPDATE` or `INSERT`, not a single SQL `ON CONFLICT` statement. This means two concurrent `POST`s for the same `(userId, provider)` can race; the `keys_user_id_provider_idx` unique index will reject the second insert but the `UPDATE` path has no such protection. The dialog disables the Save button while a request is in flight (`disabled={loading || !apiKeys[provider.id].trim()}`), so the race is rare in practice.

### `DELETE /api/api-keys?provider=…`

Removes the row matching `(userId, provider)`. Returns `{ success: true }` on any deletion (including the no-op case where no row matched). The dialog uses this endpoint for both the "Clear" button (resetting the input to empty so the user can enter a new key) and the path where the user wants to fall back to system defaults.

### Failure modes

All three handlers wrap their work in a `try/catch` that logs with `console.error(...)` and returns `{ error: 'Failed to ...' }` with status `500`. Decryption never happens on these endpoints, so the only crypto failure mode is in `encrypt(...)`: if `ENCRYPTION_KEY` is missing or not a 64-character hex string, `POST` will throw with the `openssl rand -hex 32` hint and the route will surface it as `500`. See [Encryption & Log Redaction](../concepts/encryption-and-redaction.md#failure-and-edge-case-behavior) for the full set of crypto-layer failure modes.

## HTTP surface: `/api/api-keys/check`

The route at [`app/api/api-keys/check/route.ts`](repo://app/api/api-keys/check/route.ts) is **not** a CRUD endpoint on the keys table. It answers a yes/no question the task-creation UI asks: "Given this agent and (optionally) this model, can the current user run it, and which provider is the source of the key?". It accepts two query parameters:

- `agent` (required) — one of `'claude' | 'codex' | 'copilot' | 'cursor' | 'gemini' | 'opencode'`.
- `model` (optional) — used to override the default provider when an agent supports multiple.

The handler is unauthenticated (no `getSessionFromReq` call). It always calls `getUserApiKey(provider)`, which itself falls back to `process.env.*` when there is no session — so an anonymous caller will see whatever the system env can resolve. This is intentional: the check answers "is this runnable in principle?" rather than "does the user have a key?". The handler responds with `{ success: true, hasKey, provider, agentName }`; `hasKey` is `!!apiKey`, `provider` is the resolved provider id, and `agentName` is a title-cased version of the agent parameter.

### Agent-to-provider mapping

The default mapping at the top of the route file is:

| Agent | Default provider | Override condition |
| --- | --- | --- |
| `claude` | `aigateway` | None (Claude always goes through the gateway) |
| `codex` | `aigateway` | None (Codex always goes through the gateway) |
| `copilot` | `null` | Special-cased below |
| `cursor` | `cursor` | Model name override (see below) |
| `gemini` | `gemini` | None |
| `opencode` | `openai` | Model name override (see below) |

`copilot` does not need an `OPENAI_API_KEY` etc.; it needs a GitHub token. The handler detects this case before the generic mapping and calls `getUserGitHubToken()` from `lib/github/user-token.ts`. If the user has a connected GitHub account (primary or linked via the `accounts` table), `hasKey` is `true` and `provider` is the literal string `'github'`.

For `cursor` and `opencode`, the model parameter can switch the resolved provider:

- `isAnthropicModel(model)` matches when the lowercase model name contains `'claude'`, `'sonnet'`, or `'opus'` → `provider = 'anthropic'`.
- `isGeminiModel(model)` matches when the lowercase model name contains `'gemini'` → `provider = 'gemini'`.
- `isOpenAIModel(model)` matches when the lowercase model name contains `'gpt'` or `'openai'` → `provider = 'aigateway'` (preferring the gateway over a raw `openai` key when both are present).
- For `cursor` with no recognizable pattern, the default `'cursor'` provider is kept.

`opencode` behaves the same way except the fallback is `'openai'` rather than `'cursor'` when the model name does not match any pattern. The model-name heuristics live only in this route — the agent wrappers in `lib/sandbox/agents/*.ts` do not know about them. A user who picks "Cursor" with a Claude model in the new-task UI gets an `anthropic` key check; a user who picks the same Cursor with a model string that does not match any pattern keeps the `cursor` key check.

## Resolution model: user key overlaid on system env

The two functions in [`lib/api-keys/user-keys.ts`](repo://lib/api-keys/user-keys.ts) implement a three-step precedence that is the single contract every caller relies on:

### `getUserApiKeys()`

Returns an object of the shape `{ OPENAI_API_KEY, GEMINI_API_KEY, CURSOR_API_KEY, ANTHROPIC_API_KEY, AI_GATEWAY_API_KEY }`:

1. **Initialize from `process.env.*`.** The base object is populated with the five env vars as they exist on the server. If the function is called in a context where `process.env.AI_GATEWAY_API_KEY` is unset, that field stays `undefined`.
2. **Read the session.** `getServerSession()` is called without arguments; on the route-handler path it is the same `cookies()`-backed read used elsewhere in the app. If there is no session, the function returns the env-initialized object as-is — there is no DB read at all for anonymous calls.
3. **Overlay encrypted user keys.** All rows from `keys` matching `userId` are selected in one query, each `value` is `decrypt(...)`-ed, and the matching field of the returned object is overwritten by `provider` (`openai` → `OPENAI_API_KEY`, `aigateway` → `AI_GATEWAY_API_KEY`, etc.). Unknown provider strings are silently ignored.

The DB block is wrapped in a `try/catch` that logs `'Error fetching user API keys'` and returns the env-initialized object on failure. There is no retry and no surface error; the call sites (the task routes) treat this as a non-fatal degradation.

### `getUserApiKey(provider)`

The single-provider variant used by `/api/api-keys/check`. It builds the same `systemKeys` map inline (rather than calling `getUserApiKeys`), then queries one row matching `(userId, provider)` with `.limit(1)`. If the row exists and `decrypt(value)` returns a truthy string, it is returned; otherwise the function falls back to `systemKeys[provider]`. The DB block has the same swallow-and-log error handling.

### Implications

- **A user key always wins** for the provider it is set on, regardless of whether the system env has a value for the same field. The overlay happens unconditionally after the env initialization.
- **Partial configuration is allowed.** A user with only an `aigateway` row will use their own `AI_GATEWAY_API_KEY` but the system env's `OPENAI_API_KEY` for any OpenAI-backed agent they run. There is no "use only my keys" mode.
- **Anonymous requests** are equivalent to "no user rows"; both code paths return the same env-initialized object.
- **Encryption failures degrade silently.** If `decrypt` throws on every row (e.g. `ENCRYPTION_KEY` was rotated), the catch in `getUserApiKeys` swallows the error and the user runs with system keys only. There is no signal in the response shape that anything went wrong; the only signal is the server-side `console.error('Error fetching user API keys:', error)` line.

## Dispatcher integration: pushing user keys into `process.env`

The task-creation flow needs the decrypted user keys inside the per-agent CLI processes that run *inside* the sandbox. The path is:

1. `app/api/tasks/route.ts` (POST handler) calls `getUserApiKeys()` once, before the `after(...)` callback, and threads the resulting object through `processTaskWithTimeout(...)` → `processTask(...)` → `createSandbox(...)` → `executeAgentInSandbox(...)`. The same call happens in `app/api/tasks/[taskId]/continue/route.ts` for follow-up messages.
2. The dispatcher in [`lib/sandbox/agents/index.ts`](repo://lib/sandbox/agents/index.ts#L57-L75) snapshots the original `process.env.OPENAI_API_KEY` / `GEMINI_API_KEY` / `CURSOR_API_KEY` / `ANTHROPIC_API_KEY` / `AI_GATEWAY_API_KEY` / `GH_TOKEN` / `GITHUB_TOKEN`, then assigns the user-supplied values onto `process.env` only when they are non-undefined.
3. The `try` block calls the per-agent wrapper (`executeClaudeInSandbox`, `executeCursorInSandbox`, etc.). These wrappers read `process.env.*` directly when assembling shell commands — for example `executeClaudeInSandbox` reads `process.env.AI_GATEWAY_API_KEY` to write `~/.config/claude/config.json`, and `executeCursorInSandbox` reads `process.env.CURSOR_API_KEY` to require it before launching the CLI.
4. The `finally` block restores the snapshot, so a different request landing on the same Node process after this one runs sees the original system env values.

The `finally`-restore pattern is critical because several wrappers read `process.env` rather than accepting the key as a parameter. A user key that leaks into a subsequent request's env would either (a) silently mis-attribute billing or (b) cause a Vercel sandbox launch to fail with the wrong credential.

The user-keys call is deliberately hoisted out of the `after(...)` block: inside `after`, the `getServerSession()` cookie read can race with the request lifecycle and return `undefined`, so the task route does the resolution in the original request scope and passes the snapshot through.

## Provider-to-CLI env-var mapping

The five provider rows resolve into the seven env vars the agent CLIs read. The mapping is enforced in two places (`getUserApiKeys`'s switch and `getUserApiKey`'s `systemKeys` map) and consumed by the dispatcher:

| Provider (DB enum) | Env-var name (CLI input) | Agents that read it | Where it is consumed |
| --- | --- | --- | --- |
| `anthropic` | `ANTHROPIC_API_KEY` | `opencode` | `lib/sandbox/agents/opencode.ts:263` writes `opencode auth add anthropic` with this key; `:308` propagates it as an `envVars` prefix on the `opencode run` invocation |
| `openai` | `OPENAI_API_KEY` | `codex`, `opencode` | `codex.ts:137` passes `OPENAI_API_KEY` in the version-check `env`; `opencode.ts:241-261` writes `opencode auth add openai` and propagates as an env prefix |
| `cursor` | `CURSOR_API_KEY` | `cursor` | `cursor.ts:170-177` returns `{ success: false }` with `'CURSOR_API_KEY not found'` when missing |
| `gemini` | `GEMINI_API_KEY` | `gemini` | `gemini.ts:178-189` selects `'api_key'` auth mode and copies the value into the `authEnv` it will pass to `runCommandInSandbox` |
| `aigateway` | `AI_GATEWAY_API_KEY` | `claude`, `codex`, `opencode` | `claude.ts:87` writes `~/.config/claude/config.json`; `codex.ts:79-105` validates the key prefix (`sk-` or `vck_`); `opencode.ts:31` accepts it as a fallback to `ANTHROPIC_API_KEY` |

The dispatcher does not rename providers when writing into `process.env`; the user-keys object is the same shape as the env object it overlays. The Claude wrapper additionally writes the same `AI_GATEWAY_API_KEY` into `ANTHROPIC_API_KEY` env-var prefixes on every `claude mcp add` invocation, so the Claude CLI's "ANTHROPIC_*"-named env vars also end up carrying the gateway key by transitivity — but that happens inside the sandbox, not on the host.

Copilot is the one agent whose credential does not flow through this table. It reads `process.env.GH_TOKEN` (or `GITHUB_TOKEN` as a fallback) which is set by the dispatcher from `getUserGitHubToken()` in `lib/github/user-token.ts` and sourced from the user's OAuth token (primary `users.accessToken` or the linked `accounts.accessToken` row).

## UI: the `ApiKeysDialog` component

[`components/api-keys-dialog.tsx`](repo://components/api-keys-dialog.tsx) renders a modal with five rows, one per provider. It is mounted by [`components/auth/sign-out.tsx`](repo://components/auth/sign-out.tsx#L19) and opened from the "API Keys" item in the user dropdown menu (`L138-L141`). The dialog is purely a control surface — it never sees the key values from `GET`, only the provider list.

### Provider list and placeholders

The `PROVIDERS` constant at the top of the file (`L18-L24`) defines the display order and placeholder text shown in the input:

| Display order | Provider id | Display name | Placeholder |
| --- | --- | --- | --- |
| 1 | `aigateway` | AI Gateway | `gw_…` |
| 2 | `anthropic` | Anthropic | `sk-ant-…` |
| 3 | `openai` | OpenAI | `sk-…` |
| 4 | `gemini` | Gemini | `AIza…` |
| 5 | `cursor` | Cursor | `cur_…` |

The placeholders are not validated by the API; they exist only to hint the user about the expected prefix.

### State machine

Three pieces of state drive each row:

- **`apiKeys[provider]`** — the in-progress input value. Cleared after a successful save.
- **`savedKeys`** — a `Set<Provider>` populated by the initial `GET`. Determines whether the row is rendered as "already saved" (input disabled, placeholder is `••••••••••••••••`).
- **`clearedKeys`** — a `Set<Provider>` tracking rows the user has explicitly cleared via the dialog's `handleClear` flow. Cleared from the set when the row is saved again.

The component renders one of two buttons per row based on these states:

- `showSaveButton = !hasSavedKey || isCleared` is true → the "Save" button is shown.
- Otherwise → a "Clear" button is shown, which calls `handleClear` (a `DELETE /api/api-keys?provider=…` followed by removing both `savedKeys` and `clearedKeys` from the row's sets).

The component disables the input while a request is in flight and disables the Save button when the input is empty. There is no client-side validation beyond "non-empty after `trim`"; the API does not enforce a prefix either, so a user can save any string.

### Show/hide toggle

The `Eye`/`EyeOff` icon button next to the input toggles `showKeys[provider]` for that row, switching the input `type` between `'password'` and `'text'`. This is purely a display affordance — the value is in component state and never persisted anywhere except in the eventual `POST` body.

### Error surfacing

The dialog uses Sonner toasts for both success (`'<Provider> API key saved'`) and failure (`error.error || 'Failed to save API key'`). It does not surface the dialog text on a save failure; the user must re-type the key.

## Configuration surface

The keys-management feature has no configuration surface of its own — it depends on the encryption layer (`ENCRYPTION_KEY` in `lib/crypto.ts`) and on the JWE session layer (`JWE_SECRET` in `lib/jwe/`), both of which are documented in [Encryption & Log Redaction](../concepts/encryption-and-redaction.md).

Two related operational knobs sit in the same call sites:

- **`validateEnvironmentVariables(selectedAgent, githubToken, apiKeys)`** in `lib/sandbox/config.ts` is the gate that runs before sandbox creation. It accepts an `apiKeys` object of the same shape as `getUserApiKeys()` and treats a present value in either `apiKeys` or `process.env.*` as satisfying the per-agent requirement. The check is run with the user-keys object threaded through from `getUserApiKeys()`, so a user with their own `AI_GATEWAY_API_KEY` row will pass the Claude/Codex check even when the system env does not have the key set.
- **The dispatcher mutation** in `executeAgentInSandbox` is the second gate: even if `validateEnvironmentVariables` accepts the request, the agent CLI's own precondition (e.g. `cursor.ts:170-177`) will return `{ success: false }` if `process.env.CURSOR_API_KEY` is unset at execution time. The dispatcher mutation is what keeps the user key visible to that check.

## Failure and edge-case behavior

- **Decryption failure.** A throw inside `decrypt` (e.g. `ENCRYPTION_KEY` rotated and the old rows are now unreadable) is caught in both `getUserApiKeys` and `getUserApiKey`; the function logs `'Error fetching user API keys'` and returns the env-initialized fallback. No row is decrypted successfully, so the user runs with system keys only.
- **`ENCRYPTION_KEY` missing on write.** The `POST` handler's `encrypt(apiKey)` call throws with the `openssl rand -hex 32` hint. The handler's outer catch returns `500` with `'Failed to save API key'`. The user sees a toast with the static "Failed to save API key" string and the server log carries the original crypto error.
- **Session lost mid-`after` block.** The task route fetches `getUserApiKeys()` *before* entering `after(...)`; inside `after`, the agent runs against the snapshot, not against `process.env` that the runtime might re-derive. The dispatcher's `finally` restores the snapshot so the next request does not see stale values.
- **Two concurrent `POST`s for the same provider.** The unique index `keys_user_id_provider_idx` makes the second `INSERT` fail; the JS-level upsert cannot tell whether a row was inserted or updated by another request between its `SELECT` and `INSERT`. The dialog mitigates this with a loading-state disable, but the API does not lock.
- **Provider not in the enum.** `POST` rejects an unknown provider with `400 Invalid provider`. `getUserApiKey(provider)` accepts any string but returns `systemKeys[provider]`, which is `undefined` for any provider not in the five-element map.
- **Copilot check returns `provider: 'github'`.** The check route special-cases `agent === 'copilot'` and reports `provider: 'github'`. The user-keys object shape does not contain a `github` field; the readiness check is the only place the `github` provider id is surfaced to the client.

## Extension points

- **Adding a sixth provider.** Three coordinated edits are needed: extend the `provider` enum in `lib/db/schema.ts` (both the `pgEnum` and the Zod schemas), extend the `systemKeys` map and the switch in `lib/api-keys/user-keys.ts`, extend the `PROVIDERS` array in `components/api-keys-dialog.tsx`, and add the env-var resolution to `validateEnvironmentVariables` if the new provider is required for any existing agent. A migration is required because the enum change is not a no-op for an already-deployed database.
- **Adding a seventh agent that consumes an existing provider.** The only edit is in `lib/sandbox/agents/index.ts`'s `switch` plus a new `executeXxxInSandbox` wrapper. The dispatcher mutation block already handles every existing env var, so no env-var plumbing is needed.
- **Surfacing "no key configured" before submission.** The current UI calls `/api/api-keys/check` on agent selection; an agent that resolves to `hasKey: false` shows a hint pointing the user at the API Keys dialog. Adding this UX to the home-page content component requires no API change beyond what already exists.
- **Rotating `ENCRYPTION_KEY`.** Every row in `keys` was encrypted under the old key. The application does not implement key migration — rotating `ENCRYPTION_KEY` requires every user to re-save their API key (or run a manual re-encrypt script). See [Encryption & Log Redaction → Configuration surface](../concepts/encryption-and-redaction.md#configuration-surface).
