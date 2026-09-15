---
type: operation
title: Environment Variables & Secrets
description: Catalog of every environment variable the application reads, who reads it, whether it is required or has a default, and whether the Next.js build pipeline emits it to the client (NEXT_PUBLIC_*). Documents the env-var-overlaid-by-DB-key resolution model that backs per-user API key overrides, the redaction contract enforced on every log line that could carry a secret value, and the validateEnvironmentVariables gate that runs before sandbox creation.
tags: [env, env-vars, configuration, secrets, api-keys, encryption, sandbox, vercel, postgres, auth, oauth, jwe, redaction, operations, security]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-8037e2358a2c4f9b2c722a11
    resource: repo://AGENTS.md
  - id: openwiki-source-1745d192079a6690c9cd73cc
    resource: repo://app/api/auth/github/callback/route.ts
  - id: openwiki-source-552741c58d746b8418a79f4f
    resource: repo://app/api/auth/signin/vercel/route.ts
  - id: openwiki-source-00541c8513f53ee5f55f7fa4
    resource: repo://app/api/tasks/%5BtaskId%5D/restart-dev/route.ts
  - id: openwiki-source-0ec504d4b2bc48a669486a67
    resource: repo://drizzle.config.ts
  - id: openwiki-source-819e40f1dfdc216e6306b923
    resource: repo://lib/api-keys/user-keys.ts
  - id: openwiki-source-f4a535c4711843f9246eaee1
    resource: repo://lib/auth/providers.ts
  - id: openwiki-source-d8833a44f288fa20597092dd
    resource: repo://lib/constants.ts
  - id: openwiki-source-c5e751251b730ee67d7f9fb1
    resource: repo://lib/crypto.ts
  - id: openwiki-source-56bfd247a389cd772a22dc99
    resource: repo://lib/db/client.ts
  - id: openwiki-source-526feb2334febba699169a23
    resource: repo://lib/db/schema.ts
  - id: openwiki-source-b251c9f870bdc2494ff96ac6
    resource: repo://lib/db/settings.ts
  - id: openwiki-source-8b71134f021e5bd57558cf48
    resource: repo://lib/jwe/decrypt.ts
  - id: openwiki-source-2972832ca9085b9e225f8b49
    resource: repo://lib/jwe/encrypt.ts
  - id: openwiki-source-5adc19616f446e2a3f22d17b
    resource: repo://lib/sandbox/agents/copilot.ts
  - id: openwiki-source-7563e3ff5929a46c9db6c26a
    resource: repo://lib/sandbox/agents/gemini.ts
  - id: openwiki-source-f111cb78e7ed9c79f1238982
    resource: repo://lib/sandbox/agents/index.ts
  - id: openwiki-source-78237118aac1cf9686d70d4d
    resource: repo://lib/sandbox/config.ts
  - id: openwiki-source-76eb5082498d555b89ce9862
    resource: repo://lib/sandbox/creation.ts
  - id: openwiki-source-4c0dfa7b4928caf4818e73f8
    resource: repo://lib/utils/logging.ts
  - id: openwiki-source-1bc696df8ff4275baeff50ac
    resource: repo://scripts/migrate-production.ts
  - id: openwiki-source-fdca57ca51bfa43eb5ae1066
    resource: repo://vercel-template.json
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Environment Variables & Secrets

This page is the canonical catalog of every environment variable the application reads at runtime. It records each variable's name, who reads it, whether it is required, and whether it is safe to expose to the browser. It also documents the two cross-cutting behaviors that govern how env vars are used: the **env-var-overlaid-by-DB-key** resolution model that lets individual users override system credentials with their own per-provider API keys, and the **redaction contract** in `lib/utils/logging.ts` that scrubs known credential patterns from every log line that reaches the UI.

For the AES-256-CBC and JWE primitives that turn `ENCRYPTION_KEY` and `JWE_SECRET` into the encrypted storage and session layers, see [Encryption & Log Redaction](../concepts/encryption-and-redaction.md). For the agent implementations that consume the per-provider API keys at runtime, see [Agent Implementations](../systems/agent-implementations.md). For the broader "do not log secrets, do not bake secrets into the client" rules, see [Security & Redaction](./security-and-redaction.md).

Operational tunables that drive rate limits and sandbox lifetime are exported from `lib/constants.ts`; the cross-link lives in [lib/constants.ts: tuning constants](#libconstantsts-tuning-constants). The Vercel deploy URL that lists the required-var set for a fresh `Deploy with Vercel` click is also defined there.

## Resolution model: env vars overlaid by per-user keys

Every per-provider API key (`OPENAI_API_KEY`, `GEMINI_API_KEY`, `CURSOR_API_KEY`, `ANTHROPIC_API_KEY`, `AI_GATEWAY_API_KEY`) follows the same three-tier resolution model. The two relevant functions are `getUserApiKeys()` and `getUserApiKey(provider)` in `lib/api-keys/user-keys.ts`:

1. **System fallback.** The function initializes an object from `process.env.*`. If there is no session (anonymous request), this object is returned as-is.
2. **DB overlay.** If a session exists, the function reads every row from the `keys` table for the signed-in user, calls `decrypt(key.value)` (AES-256-CBC, decrypted with `ENCRYPTION_KEY`), and overwrites the matching field of the env-initialized object by provider. Unknown provider strings are ignored.
3. **System fallback on failure.** If the DB read throws, the function logs a static `'Error fetching user API keys'` and returns the env-initialized object. There is no retry and no surface error.

Because the decrypted user keys overwrite the system keys in the same object, the dispatcher's downstream code never knows whether the credential came from the environment or the user's profile. This is the contract that lets the rest of the codebase treat API keys as a single uniform source.

The dispatcher mirrors this idea one level deeper for the sandbox agents. `executeAgentInSandbox` in `lib/sandbox/agents/index.ts` snapshots the original `process.env.*` values for all six provider keys plus `GH_TOKEN`/`GITHUB_TOKEN`, then mutates `process.env` with the user-supplied keys for the duration of the agent's `try` block, and **always** restores the original values in a `finally` block. This matters because several agent implementations (`lib/sandbox/agents/codex.ts`, `cursor.ts`, `opencode.ts`) read `process.env.*` directly when assembling shell commands, rather than receiving the key as a parameter.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    subgraph Defaults["Step 1: System defaults"]
      E["process.env.*<br/>OPENAI_API_KEY etc."]
      B["apiKeys object<br/>initialized from env"]
      E --> B
    end
    subgraph DB["Step 2: DB overlay (authenticated only)"]
      S["getServerSession()"]
      Q["keys table<br/>rows WHERE userId = session.user.id"]
      D["decrypt(key.value)<br/>via ENCRYPTION_KEY"]
      B2["apiKeys object<br/>overwritten by provider"]
      S --> Q
      Q --> D
      D --> B2
    end
    subgraph Run["Step 3: Agent execution"]
      I["executeAgentInSandbox<br/>snapshot process.env"]
      R["per-agent process.env<br/>reads (codex, cursor, …)"]
      F["finally: restore original env"]
      I --> R --> F
    end
    B --> B2
    B2 --> Run
```

*Diagram: env-var resolution and overlay. Step 1 produces a uniform credential object; step 2 overwrites per-provider fields with decrypted user keys when a session is present; step 3 lets sandbox agents read `process.env` directly while a `try/finally` guarantees the original system values are restored before the function returns.*

## Master catalog

The table below lists every environment variable the application reads. **Client-safe** means the value is exposed to the browser via Next.js's `NEXT_PUBLIC_` prefix. **Required** means the application throws or refuses to start when the variable is unset; **optional** means the variable has a default, is only consumed by a code path that is itself optional, or is ignored when absent.

| Name | Primary reader(s) | Required | Client-safe |
| --- | --- | --- | --- |
| `ANTHROPIC_API_KEY` | `lib/api-keys/user-keys.ts`, `lib/sandbox/agents/opencode.ts`, `lib/sandbox/config.ts` | Optional (required for the `opencode` agent unless `AI_GATEWAY_API_KEY` is also set) | No |
| `OPENAI_API_KEY` | `lib/api-keys/user-keys.ts`, `lib/sandbox/agents/codex.ts`, `lib/sandbox/agents/opencode.ts`, `lib/sandbox/agents/index.ts` | Optional (required for `codex`/`opencode` if no `AI_GATEWAY_API_KEY` fallback) | No |
| `GEMINI_API_KEY` | `lib/api-keys/user-keys.ts`, `lib/sandbox/agents/gemini.ts`, `lib/sandbox/agents/index.ts` | Optional (required for the `gemini` agent unless a Google Cloud auth path is used) | No |
| `CURSOR_API_KEY` | `lib/api-keys/user-keys.ts`, `lib/sandbox/agents/cursor.ts`, `lib/sandbox/agents/index.ts` | Optional (required for the `cursor` agent) | No |
| `AI_GATEWAY_API_KEY` | `lib/api-keys/user-keys.ts`, `lib/sandbox/agents/claude.ts`, `lib/sandbox/agents/codex.ts`, `lib/sandbox/agents/index.ts`, `lib/sandbox/config.ts`, `lib/utils/branch-name-generator.ts`, `lib/utils/commit-message-generator.ts`, `lib/utils/title-generator.ts` | Optional (required for `claude` and `codex` agents and for the three generators) | No |
| `GH_TOKEN` / `GITHUB_TOKEN` | `lib/sandbox/agents/copilot.ts`, `lib/sandbox/agents/index.ts` | Optional (required for the `copilot` agent) | No |
| `GOOGLE_API_KEY` | `lib/sandbox/agents/gemini.ts` | Optional (alternative Gemini auth path) | No |
| `GOOGLE_GENAI_USE_VERTEXAI` | `lib/sandbox/agents/gemini.ts` | Optional (must be set together with `GOOGLE_API_KEY` to select Vertex AI auth) | No |
| `GOOGLE_CLOUD_PROJECT` | `lib/sandbox/agents/gemini.ts` | Optional (alternative Gemini auth path) | No |
| `SANDBOX_VERCEL_TOKEN` | `lib/sandbox/config.ts`, `lib/sandbox/creation.ts`, `app/api/tasks/[taskId]/discard-file-changes/route.ts`, `app/api/tasks/[taskId]/file-content/route.ts`, `lib/utils/logging.ts` | Required (sandbox creation refuses to start without it) | No |
| `SANDBOX_VERCEL_TEAM_ID` | Same readers as `SANDBOX_VERCEL_TOKEN` | Required | No |
| `SANDBOX_VERCEL_PROJECT_ID` | Same readers as `SANDBOX_VERCEL_TOKEN` | Required | No |
| `GITHUB_CLIENT_SECRET` | `app/api/auth/github/callback/route.ts`, `app/api/auth/signout/route.ts` | Required (if GitHub auth is enabled) | No |
| `VERCEL_CLIENT_SECRET` | `app/api/auth/signin/vercel/route.ts`, `app/api/auth/callback/vercel/route.ts`, `app/api/auth/signout/route.ts` | Required (if Vercel auth is enabled) | No |
| `JWE_SECRET` | `lib/jwe/encrypt.ts`, `lib/jwe/decrypt.ts` | Required (cookie encryption throws `'Missing JWE secret'` if unset) | No |
| `ENCRYPTION_KEY` | `lib/crypto.ts` | Required (must be a 64-character hex string; both `encrypt` and `decrypt` throw otherwise) | No |
| `POSTGRES_URL` | `drizzle.config.ts`, `lib/db/client.ts`, `scripts/migrate-production.ts` | Required at runtime; required by `pnpm db:migrate` and `pnpm db:push` | No |
| `MAX_MESSAGES_PER_DAY` | `lib/constants.ts`, `lib/db/settings.ts` | Optional (default `5`) | No |
| `MAX_SANDBOX_DURATION` | `lib/constants.ts`, `lib/db/schema.ts`, `lib/db/settings.ts` | Optional (default `300` minutes) | No |
| `NODE_ENV` | `app/api/auth/github/signin/route.ts`, `app/api/auth/signin/github/route.ts`, `app/api/auth/signin/vercel/route.ts`, `lib/session/create-github.ts`, `lib/session/create.ts` | Set by Next.js | No |
| `VERCEL_ENV` | `scripts/migrate-production.ts` | Set by Vercel | No |
| `NEXT_PUBLIC_AUTH_PROVIDERS` | `lib/auth/providers.ts` | Optional (default `github`) | Yes |
| `NEXT_PUBLIC_GITHUB_CLIENT_ID` | `app/api/auth/github/callback/route.ts`, `app/api/auth/github/signin/route.ts`, `app/api/auth/signin/github/route.ts`, `app/api/auth/signout/route.ts`, `components/home-page-content.tsx` | Required if GitHub auth is enabled | Yes |
| `NEXT_PUBLIC_VERCEL_CLIENT_ID` | `app/api/auth/signin/vercel/route.ts`, `app/api/auth/callback/vercel/route.ts`, `app/api/auth/signout/route.ts` | Required if Vercel auth is enabled | Yes |

The two `NEXT_PUBLIC_*` rows are the only variables Next.js inlines into the browser bundle. Everything else is server-only and must never be prefixed with `NEXT_PUBLIC_` or it will be readable in the static HTML payload and in the client JS chunk.

## Per-category breakdown

### Auth providers (`lib/auth/providers.ts`)

`NEXT_PUBLIC_AUTH_PROVIDERS` is parsed by `getEnabledAuthProviders` into a `{ github: boolean, vercel: boolean }` record. The function splits the comma-separated string, lowercases each entry, and tests for membership in a fixed allowlist of `github` and `vercel`. Anything outside that set is silently ignored; an unset variable defaults to `github` only. The record is consumed by the sign-in UI to decide which buttons to render.

The two OAuth client IDs that pair with `NEXT_PUBLIC_AUTH_PROVIDERS` are client-exposed by Next.js convention (`NEXT_PUBLIC_GITHUB_CLIENT_ID` and `NEXT_PUBLIC_VERCEL_CLIENT_ID`). Both client secrets (`GITHUB_CLIENT_SECRET` and `VERCEL_CLIENT_SECRET`) are strictly server-only and are read inside route handlers that exchange the OAuth code or revoke the token.

### Encryption keys (`lib/crypto.ts`, `lib/jwe/`)

`ENCRYPTION_KEY` is the AES-256-CBC at-rest key. The helper `getEncryptionKey()` decodes it from hex, rejects any value that is not exactly 32 bytes (64 hex characters), and returns `null` when unset. Both `encrypt()` and `decrypt()` throw a static error pointing to `openssl rand -hex 32` when the key is missing or malformed.

`JWE_SECRET` is the symmetric key for the A256GCM session cookie (`alg: 'dir'`). Both `lib/jwe/encrypt.ts` and `lib/jwe/decrypt.ts` default their `secret` argument to `process.env.JWE_SECRET`. The encrypt path throws `'Missing JWE secret'` when the variable is unset; the decrypt path also throws the same string rather than silently degrading, although a tampered ciphertext would already have been swallowed by the `try/catch` that wraps `jwtDecrypt`. The two secrets are deliberately different keys with different formats — rotating one does not affect the other.

### Database (`lib/db/client.ts`, `scripts/migrate-production.ts`)

`POSTGRES_URL` is the only database connection string. It is required by `lib/db/client.ts` (which throws `'POSTGRES_URL environment variable is required'` and never falls back to a default), by `drizzle.config.ts` for any live `pnpm db:migrate` or `pnpm db:push` run, and by `scripts/migrate-production.ts` after the `VERCEL_ENV === 'production'` guard. In production deployments the Neon integration declared in `vercel-template.json` provisions this variable automatically; locally the developer is expected to set it via the shell.

### Sandbox runtime (`lib/sandbox/config.ts`, `lib/sandbox/creation.ts`)

The three `SANDBOX_VERCEL_*` variables are the hard prerequisites for creating a Vercel sandbox. `validateEnvironmentVariables` in `lib/sandbox/config.ts` checks all three up front and produces a single comma-joined error message when any is missing; `createSandbox` in `lib/sandbox/creation.ts` then passes the trio to the `Sandbox.create(...)` call alongside `timeout`, `ports`, `runtime`, and `resources`. The same three variables are also read by the per-task routes that need to rehydrate a sandbox (`/api/tasks/[taskId]/discard-file-changes`, `/api/tasks/[taskId]/file-content`, `/api/tasks/[taskId]/restart-dev`).

The deploy URL in `lib/constants.ts` (see [lib/constants.ts: tuning constants](#libconstantsts-tuning-constants)) encodes `SANDBOX_VERCEL_TEAM_ID`, `SANDBOX_VERCEL_PROJECT_ID`, `SANDBOX_VERCEL_TOKEN`, `JWE_SECRET`, and `ENCRYPTION_KEY` as the required `env=` parameters of the "Deploy with Vercel" link, so a fresh clone from that URL is forced to set those five before the build can succeed.

### Per-agent API keys

The five agent keys (`OPENAI_API_KEY`, `GEMINI_API_KEY`, `CURSOR_API_KEY`, `ANTHROPIC_API_KEY`, `AI_GATEWAY_API_KEY`) are read in three layers:

- **DB-encrypted overlay** via `getUserApiKeys()` / `getUserApiKey(provider)` in `lib/api-keys/user-keys.ts`.
- **Dispatcher mutation** via `executeAgentInSandbox` in `lib/sandbox/agents/index.ts`, which temporarily assigns the user-supplied keys to `process.env.*` so per-agent implementations can read them directly.
- **Per-agent reads** inside each `lib/sandbox/agents/<name>.ts` module when it assembles shell environment prefixes, runs `claude auth login`, or writes the key into an `auth add` pipe.

`validateEnvironmentVariables` performs an additional check: it rejects a task before sandbox creation when the selected agent's required key is missing from both the user overlay and the system env. The mapping is:

| Selected agent | Required key | Notes |
| --- | --- | --- |
| `claude` | `AI_GATEWAY_API_KEY` | Sole path; no fallback |
| `codex` | `AI_GATEWAY_API_KEY` | Sole path; no fallback |
| `gemini` | `GEMINI_API_KEY` | Can be supplied via `GOOGLE_API_KEY`+`GOOGLE_GENAI_USE_VERTEXAI` or `GOOGLE_CLOUD_PROJECT` instead; the env-var requirement is only checked against `GEMINI_API_KEY` |
| `cursor` | `CURSOR_API_KEY` | Sole path; no fallback |
| `opencode` | `AI_GATEWAY_API_KEY` *or* `ANTHROPIC_API_KEY` | Either is accepted |
| `copilot` | `GH_TOKEN` or `GITHUB_TOKEN` | Validated inside the agent itself, not by `validateEnvironmentVariables` |

In addition to the agent dispatch path, three server-side generators require `AI_GATEWAY_API_KEY` and throw when it is unset: `lib/utils/branch-name-generator.ts`, `lib/utils/commit-message-generator.ts`, and `lib/utils/title-generator.ts`.

### `lib/constants.ts`: tuning constants

The following constants are exported from `lib/constants.ts` and are sourced from `process.env` with hardcoded defaults. None of them are secrets; none are client-safe (they are server-only `number` values).

| Constant | Env var | Default | Reader(s) |
| --- | --- | --- | --- |
| `MAX_MESSAGES_PER_DAY` | `MAX_MESSAGES_PER_DAY` | `5` | `lib/constants.ts`, `lib/db/settings.ts` |
| `MAX_SANDBOX_DURATION` | `MAX_SANDBOX_DURATION` | `300` (minutes) | `lib/constants.ts`, `lib/db/schema.ts` (column default), `lib/db/settings.ts` |

`MAX_MESSAGES_PER_DAY` controls the per-user daily message rate limit. `MAX_SANDBOX_DURATION` is the per-sandbox maximum lifetime in minutes; the value also seeds the `tasks.max_duration` column default in `lib/db/schema.ts` so that newly created tasks inherit the env-var-driven ceiling without needing a separate migration.

`VERCEL_DEPLOY_URL` and `VERCEL_DEPLOY_BUTTON_URL` are also exported from `lib/constants.ts`. They are not environment variables — they are hardcoded `https://vercel.com/new/clone?…` URLs whose `env=` parameter lists the required variables for a fresh clone. The `VERCEL_DEPLOY_URL` string itself encodes `SANDBOX_VERCEL_TEAM_ID`, `SANDBOX_VERCEL_PROJECT_ID`, `SANDBOX_VERCEL_TOKEN`, `JWE_SECRET`, and `ENCRYPTION_KEY` as the required set, so a README "Deploy with Vercel" button never produces a half-configured instance.

### `GH_TOKEN` vs `GITHUB_TOKEN`

The Copilot agent reads both names and prefers `GH_TOKEN`, falling back to `GITHUB_TOKEN` if the former is unset. The dispatcher's `process.env` snapshot/restore block treats them as a pair, so any mutation of one during the agent's lifetime restores both on exit. The pair is never logged; both names are in the redaction regex in `lib/utils/logging.ts` (`GH_TOKEN` matches the `GITHUB_TOKEN` pattern because the regex begins with `GITHUB_TOKEN`).

## Client-safe variables: the `NEXT_PUBLIC_` boundary

Next.js only inlines variables whose names start with `NEXT_PUBLIC_` into the browser bundle. The application uses exactly three of them, and each has a tightly scoped consumer:

- **`NEXT_PUBLIC_AUTH_PROVIDERS`** — parsed in `lib/auth/providers.ts` to decide which sign-in buttons to render. Defaults to `github`. Anything other than `github` or `vercel` is silently dropped.
- **`NEXT_PUBLIC_GITHUB_CLIENT_ID`** — read in five places: the GitHub OAuth start route, the GitHub OAuth callback, the legacy GitHub sign-in routes, the sign-out route (as part of the Basic-auth header), and the home-page content component that renders the sign-in button.
- **`NEXT_PUBLIC_VERCEL_CLIENT_ID`** — read in three places: the Vercel PKCE sign-in start, the Vercel callback, and the sign-out revoke call.

A common pitfall: `NEXT_PUBLIC_GITHUB_CLIENT_ID` is also used inside `app/api/auth/github/callback/route.ts` as if it were a server-only secret, but the client-side bundle already exposes the same value, so there is no additional leakage. The companion server-only secrets (`GITHUB_CLIENT_SECRET`, `VERCEL_CLIENT_SECRET`) are not prefixed with `NEXT_PUBLIC_` and must stay that way.

Adding a new client-safe variable requires the `NEXT_PUBLIC_` prefix; adding a new server-only secret requires the opposite discipline — confirming that no prefix is set so Next.js leaves the value out of the bundle. AGENTS.md's *Configuration Security → Environment Variables* section codifies this as a "never expose to the client" list (Vercel sandbox credentials, LLM API keys, GitHub tokens, `JWE_SECRET`, `ENCRYPTION_KEY`, and any user-supplied API key).

## Log redaction contract

The redaction contract has two halves and they are enforced by different layers:

1. **Primary defense: static log strings.** AGENTS.md's *Critical: No Dynamic Values in Logs* section forbids any template-literal logging that interpolates a variable, so a secret value should never appear in a log line in the first place. This applies to `logger.info/error/success/command`, `console.log`, `console.error`, and `console.warn`.
2. **Backup defense: `redactSensitiveInfo`.** `lib/utils/logging.ts` exports `redactSensitiveInfo(message: string): string`, which applies a series of regex patterns and replaces every matched credential value with a masked form (`first-4` + `*` × `max(8, len-8)` + `last-4`, or full stars for short values). The patterns cover `ANTHROPIC_API_KEY=sk-ant-…`, `OPENAI_API_KEY=sk-…`, `GITHUB_TOKEN=gh[phosr]_…`, GitHub-URL-embedded tokens, `Bearer …`, generic `*_KEY`/`*_TOKEN`/`*_SECRET`/`*_PASSWORD`/`*_TEAM_ID`/`*_PROJECT_ID` assignments, and the three `SANDBOX_VERCEL_*` variables. The function is invoked from `createLogEntry` (every persisted log line passes through it) and is also called directly by `lib/sandbox/creation.ts` and `lib/sandbox/agents/codex.ts` before echoing shell command lines or command output into the user-facing logger.

A single known gap: `redactSensitiveInfo` will not scrub a secret that is *not* in its regex list. For example, `POSTGRES_URL` is not in the pattern set, so a connection-string log line that includes a password would leak. The defense against that scenario is purely the static-string logging rule, not the regex.

## Required-vs-optional entrypoint: `validateEnvironmentVariables`

The single chokepoint that converts "env vars present" into a "ready to create a sandbox" decision is `validateEnvironmentVariables(selectedAgent, githubToken, apiKeys?)` in `lib/sandbox/config.ts`. It returns `{ valid: boolean, error?: string }` and accumulates errors so the caller can render one combined message. The set of checks, in order, is:

1. Per-agent key requirement (see the table in [Per-agent API keys](#per-agent-api-keys)), where each requirement accepts either the supplied `apiKeys` object or the corresponding `process.env.*`.
2. A GitHub token — supplied via `githubToken` (the user's OAuth token, not an env var) — required for repository access. The error message points the user to the connect-Git-account flow.
3. The three `SANDBOX_VERCEL_*` variables, each checked independently against `process.env`.

`createSandbox` calls `validateEnvironmentVariables` before any sandbox allocation, logs a static `'Environment variables validated'`, and rethrows the combined error string when validation fails. A misconfigured deploy therefore fails at the gate rather than at the Vercel API call, which would produce a less actionable error.

## Adding or modifying environment variables

The procedural rules below are the minimum set needed to keep the catalog, the redaction contract, and AGENTS.md in sync when the set of variables changes:

1. **Pick the right boundary.** If the value must reach the browser, prefix with `NEXT_PUBLIC_`. If it must not, do not.
2. **Catalog it.** Add a row to the [Master catalog](#master-catalog) above, name the primary reader(s), and mark it required or optional with a default.
3. **Wire a redaction pattern** in `lib/utils/logging.ts` if the new variable carries a secret value, and add the variable name to AGENTS.md's *Environment Variables → Never expose* list. The redaction regex's variable-name branch (`[A-Z_]*(?:KEY|TOKEN|SECRET|PASSWORD|TEAM_ID|PROJECT_ID)[A-Z_]*`) already covers most credentials, so this step is usually only needed for unusual names or for `SANDBOX_VERCEL_*` style prefix-based identifiers.
4. **Validate at the gate.** If the variable is required for any flow, extend `validateEnvironmentVariables` (or the per-agent check inside the relevant `lib/sandbox/agents/*.ts` module) so a misconfigured deploy fails with a single actionable error rather than a partial sandbox failure.
5. **Update the Vercel deploy URL.** If the new variable is required at clone time, add it to the `env=` parameter in `VERCEL_DEPLOY_URL` in `lib/constants.ts`. The URL-encoded list currently contains the five canonical required variables; adding a sixth requires regenerating the URL-encoded form.
