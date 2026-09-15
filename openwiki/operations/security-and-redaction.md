---
type: operation
title: Security Policy & Log Redaction Rules
description: The two-layer defense that keeps secrets out of user-visible logs - the static-string rule enforced in code review and the redactSensitiveInfo regex pass that scrubs known credential patterns from any sandbox output that reaches the UI - plus the env-var never-expose list and the client-safe (NEXT_PUBLIC_) boundary.
tags: [security, redaction, logging, secrets, credentials, vercel, anthropic, openai, github, sandbox, operations]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-8037e2358a2c4f9b2c722a11
    resource: repo://AGENTS.md
  - id: openwiki-source-a02d819c42a405c6b6116e8f
    resource: repo://lib/sandbox/agents/claude.ts
  - id: openwiki-source-bbbc9cdf967cf52b393b8240
    resource: repo://lib/sandbox/agents/opencode.ts
  - id: openwiki-source-76eb5082498d555b89ce9862
    resource: repo://lib/sandbox/creation.ts
  - id: openwiki-source-4c0dfa7b4928caf4818e73f8
    resource: repo://lib/utils/logging.ts
  - id: openwiki-source-553b6b82f8719a4f1f3940bf
    resource: repo://lib/utils/task-logger.ts
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Security Policy & Log Redaction Rules

The application handles enough sensitive material - OAuth tokens, user-supplied LLM API keys, Vercel sandbox credentials, GitHub access tokens - that a single leaky `console.log` would expose them in the UI. The defense against that is two layers, enforced by different mechanisms, and both are codified in `AGENTS.md` and `lib/utils/logging.ts`. This page describes the policy, the static-string rule, the regex redaction layer, the secrets list that must never appear in logs, and the client-safe variable boundary.

For the cryptographic primitives that protect secrets at rest (AES-256-CBC) and the JWE A256GCM session cookie, see [Encryption & Log Redaction](../concepts/encryption-and-redaction.md). For the environment-variable catalog that defines which secrets exist and who reads them, see [Environment Variables & Secrets](./environment-variables.md). For the GitHub OAuth flow that produces the user tokens the redaction layer is designed to protect, see [GitHub OAuth](../integrations/github-oauth.md).

## Layer 1: the static-string rule (primary defense)

`AGENTS.md` declares, under *Security Rules → CRITICAL: No Dynamic Values in Logs*, that every log statement must be a static string. There are no exceptions and no severity-based escape hatch - `info`, `error`, `success`, `command`, `console.log`, `console.error`, and `console.warn` are all covered.

The rule is the project's first line of defense because logs are streamed directly to the UI. The `tasks.logs` column in Postgres is `jsonb` and the task-detail page polls it; whatever an agent file writes via `TaskLogger.append` ends up in the browser bundle after the redaction pass. A template literal such as ``logger.info(`Task created: ${taskId}`)`` would put a task ID into a page that any signed-in user can view, and worse, a template literal such as ``logger.error(`Error for ${provider}:`, error)`` could surface a credential that the redaction regex happened to miss. The static-string rule eliminates the entire class of bug.

The good-vs-bad pair from `AGENTS.md` is the canonical reference:

```typescript
// BAD - Contains dynamic values
await logger.info(`Task created: ${taskId}`)
await logger.error(`Failed to process ${filename}`)
console.log(`User ${userId} logged in`)
console.error(`Error for ${provider}:`, error)

// GOOD - Static strings only
await logger.info('Task created')
await logger.error('Failed to process file')
console.log('User logged in')
console.error('Error occurred:', error)
```

Two practical adaptations follow:

- **Server-side debugging.** `console.error('Sandbox creation error:', error)` is allowed because the variable being logged is the `Error` object itself, whose `message` and `stack` are useful for triage and never reach the UI (only `TaskLogger` writes to the user-visible `tasks.logs` column). The AGENTS.md guidance is to still avoid credentials in server-side logs.
- **Progress messages.** `logger.updateProgress(50, 'Installing dependencies')` is allowed because both arguments are static; the progress number is a parameter, not an interpolated log string.

The same rule is enforced by the *Logging Best Practices* section with three concrete patterns: log the *action* (`'Sandbox created successfully'`) rather than the value, use `console.error` for server-side debugging, and pass a static progress message to `updateProgress`.

## Sensitive data classes that must never appear in logs

`AGENTS.md` lists the data classes that must never be interpolated into a log string, regardless of severity. The list is the canonical contract for what the redaction regex is trying to defend:

| Class | Examples | Why |
| --- | --- | --- |
| Vercel credentials | `SANDBOX_VERCEL_TOKEN`, `SANDBOX_VERCEL_TEAM_ID`, `SANDBOX_VERCEL_PROJECT_ID` | Grants the holder the ability to create and inspect any sandbox in the linked Vercel project |
| User IDs and PII | Internal user IDs, OAuth `externalId`, email addresses | Ties a log line back to a specific account |
| File paths and repository URLs | `/Users/alice/...`, `https://github.com/<owner>/<repo>` | Reveals server filesystem layout and customer project names |
| Branch names and commit messages | `feature/alice-fix`, `WIP: temp` | Reveals in-flight work and customer code |
| Error details | `error.message`, `error.stack` that may embed a credential | Errors frequently echo the input that triggered them, including secrets |
| Any dynamic value that exposes system internals | Anything not in the static-string allowlist | Catch-all that backs the rule |

`AGENTS.md` reinforces the list in two more places. The *Configuration Security → Environment Variables* section enumerates the secret-bearing variables by name (`SANDBOX_VERCEL_TOKEN`, `SANDBOX_VERCEL_TEAM_ID`, `SANDBOX_VERCEL_PROJECT_ID`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `CURSOR_API_KEY`, `GH_TOKEN`/`GITHUB_TOKEN`, `JWE_SECRET`, `ENCRYPTION_KEY`, and any user-provided API key) and asserts they must never be exposed in logs or to the client. The *Compliance Checklist* requires that before submission there are no template literals with `${}` in any log statement and no sensitive data in error messages.

A related but separate rule is the *Error Handling* section: generic messages to users (`logger.error('Operation failed')`) and detailed debugging only on the server (`console.error('Detailed error for debugging:', error)`). The UI never receives `error.message` directly because the only error string persisted to `tasks.logs` is the one passed to `logger.error`, which AGENTS.md requires to be static.

## Layer 2: `redactSensitiveInfo` (backup defense)

`redactSensitiveInfo(message: string): string` in `lib/utils/logging.ts` is the second layer. Per `AGENTS.md` it is explicitly described as a "backup measure only"; the primary defense is the static-string rule above. The function exists because real sandbox output - shell command lines, package-manager stdout, agent stderr - contains dynamic values by construction, and a regex pass is the only practical way to keep those strings from leaking credentials.

### What `redactSensitiveInfo` matches

The function runs three passes over the input string. Every pass preserves the variable name and replaces the value with `first-4` + `*` × `max(8, len - 8)` + `last-4` (or full stars if the value is eight characters or shorter). The result is that a leak reveals *which* key leaked but only the first four and last four characters of the value.

1. **Provider-specific API key patterns.** `ANTHROPIC_API_KEY=<sk-ant-…>`, `OPENAI_API_KEY=<sk-…>`, `GITHUB_TOKEN=<ghp_/gho_/ghu_/ghs_/ghr_…>`, `https://<token>(:x-oauth-basic)?@github.com` (GitHub auth-in-URL form, special-cased to preserve the URL structure), generic `API_KEY=<value>`, `Bearer <token>`, generic `TOKEN=<value>`, and the three `SANDBOX_VERCEL_*` variables.
2. **JSON field pattern.** `"teamId"` and `"projectId"` keys (case-insensitive) with their string values replaced by `"[REDACTED]"`. This catches Vercel sandbox config blobs such as `{ teamId: "team_abc123", projectId: "prj_xyz" }`.
3. **Environment-variable catch-all.** Any uppercase name ending in `KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `TEAM_ID`, or `PROJECT_ID` (with optional leading/trailing underscores) followed by an alphanumeric value of at least 8 characters. This is what scrubs inline `OPENAI_API_KEY=sk-ant-***` strings an agent CLI may echo back even when pass 1 misses them.

Pass 1 runs first because its patterns are more specific; pass 3 catches everything pass 1 missed.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    subgraph Input["Input string"]
      S1["shell command line"]
      S2["package-manager stdout"]
      S3["agent stderr"]
    end
    subgraph Pass["redactSensitiveInfo"]
      P1["Pass 1: provider patterns<br/>sk-ant-, sk-, gh[pousr]_, Bearer, SANDBOX_VERCEL_*"]
      P2["Pass 2: JSON field pattern<br/>teamId / projectId"]
      P3["Pass 3: env-var catch-all<br/>*_KEY / *_TOKEN / *_SECRET / *_PASSWORD / *_TEAM_ID / *_PROJECT_ID"]
      S1 --> P1 --> P2 --> P3
      S2 --> P1
      S3 --> P1
    end
    Output["Masked output<br/>variable name + first4/last4"]
    P3 --> Output
```

*Diagram: the three passes inside `redactSensitiveInfo`. Each pass replaces matched values with the first-4 / last-4 mask while preserving the variable name.*

### Where `redactSensitiveInfo` runs

The function is invoked from `createLogEntry(type, message, timestamp?)` in `lib/utils/logging.ts`, which is the helper that `TaskLogger.append` (`lib/utils/task-logger.ts`) and the convenience methods `info` / `command` / `error` / `success` use for every log line. Because every persisted entry in `tasks.logs` passes through `createLogEntry`, every user-visible log line is redacted at least once.

Defense in depth: the per-agent `runAndLogCommand` helpers also call `redactSensitiveInfo` directly on the shell command, stdout, and stderr *before* the value reaches `logger.command` / `logger.info` / `logger.error`. The exact sites are:

- `lib/sandbox/creation.ts` - the bootstrap helper used during sandbox setup and `pnpm install` / `pnpm build` invocations.
- `lib/sandbox/agents/claude.ts`, `codex.ts`, `copilot.ts`, `cursor.ts`, `gemini.ts`, `opencode.ts` - each agent implementation has its own `runAndLogCommand` that redacts the same three strings.

Two agent files have additional redaction logic for command-line-shaped secrets that the broad regex would miss:

- `lib/sandbox/agents/claude.ts` calls `fullCommand.replace(aiGatewayKey, '[REDACTED]')` for the Claude CLI launch command, where `aiGatewayKey` is the literal `AI_GATEWAY_API_KEY` value pulled from the env.
- `lib/sandbox/agents/opencode.ts` calls `fullCommand.replace(/API_KEY="[^"]*"/g, 'API_KEY="[REDACTED]"')` to scrub every inline `API_KEY="…"` prefix in the OpenCode launch command, because that command inlines multiple `API_KEY="…"` pairs as environment prefixes.

`opencode.ts` then runs the broader `redactSensitiveInfo(stdout)` and `redactSensitiveInfo(stderr)` passes over the agent's output, so a credential that escapes one layer is caught by the next.

### What `redactSensitiveInfo` does not catch

`AGENTS.md` calls the regex a backup measure for a reason: it is a closed allowlist of patterns. Any secret that is not in the regex list reaches the UI verbatim. The concrete gap is `POSTGRES_URL`, which is not in the pattern set - a connection-string log line that includes a password would leak, and the only defense is the static-string rule. The same is true for any future secret that ships before its redaction pattern lands.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    A["Agent code:<br/>logger.* / console.* call"] --> B{"Static string?"}
    B -- "yes" --> C["Passes to redactSensitiveInfo<br/>(no-op for the value)"]
    B -- "no" --> D["Compliance failure<br/>AGENTS.md rule violated"]
    C --> E["createLogEntry<br/>redacts + persists"]
    E --> F["tasks.logs<br/>jsonb column"]
    F --> G["Streamed to UI"]
    H["Sandbox command / stdout / stderr"] --> I["redactSensitiveInfo at call site"]
    I --> J["logger.command/info/error"]
    J --> E
```

*Diagram: the two entrypoints into `redactSensitiveInfo`. Agent-authored logs must be static strings; sandbox-derived logs are scrubbed by the regex pass. The static-string rule is what closes the gap the regex cannot.*

## Verifying the static-string rule

`AGENTS.md` codifies two `grep` commands under *Testing Changes* that detect dynamic values in user-facing logs:

```bash
# Check for logger statements with template literals
grep -r "logger\.(info|error|success|command)\(\`.*\$\{" .

# Check for console statements with template literals
grep -r "console\.(log|error|warn|info)\(\`.*\$\{" .
```

These return every logger / console call that uses a template literal, which by AGENTS.md is a compliance failure. The *Compliance Checklist* repeats the test as a series of pre-submission gates:

- No template literals with `${}` in any log statements.
- All `logger` calls use static strings.
- All `console` calls use static strings (for user-facing logs).
- No sensitive data in error messages.
- Test in the UI to confirm no data leakage.
- Server-side debugging logs don't expose credentials.

Together with the broader hygiene checks (`pnpm format`, `pnpm type-check`, `pnpm lint`, `pnpm build`), the checklist is the operational form of the static-string rule.

## Configuration security: env-var exposure rules

`AGENTS.md` *Configuration Security → Environment Variables* is the policy half of the secrets catalog. It forbids logging or client-exposing any of the variables below:

- `SANDBOX_VERCEL_TOKEN` - Vercel API token
- `SANDBOX_VERCEL_TEAM_ID` - Vercel team identifier
- `SANDBOX_VERCEL_PROJECT_ID` - Vercel project identifier
- `ANTHROPIC_API_KEY` - Anthropic / Claude API key
- `OPENAI_API_KEY` - OpenAI API key
- `GEMINI_API_KEY` - Google Gemini API key
- `CURSOR_API_KEY` - Cursor API key
- `GH_TOKEN` / `GITHUB_TOKEN` - GitHub personal access token
- `JWE_SECRET` - session cookie encryption secret
- `ENCRYPTION_KEY` - at-rest encryption key
- Any user-provided API key

The companion rule is *Client-Safe Variables*: only `NEXT_PUBLIC_AUTH_PROVIDERS` and `NEXT_PUBLIC_GITHUB_CLIENT_ID` are allowed to carry the `NEXT_PUBLIC_` prefix, because that prefix tells Next.js to inline the value into the browser bundle. Every other variable must be unprefixed so it stays server-only. The full env-var catalog, including the two `NEXT_PUBLIC_*` rows and the `VERCEL_DEPLOY_URL` button parameterization that lists the required variables for a fresh clone, is in [Environment Variables & Secrets](./environment-variables.md).

## Log entry helpers built on the redaction layer

`lib/utils/logging.ts` exports four thin convenience helpers on top of `createLogEntry`:

- `createInfoLog(message)` → `{ type: 'info', message: redactSensitiveInfo(message), timestamp }`
- `createCommandLog(command, args?)` → joins `command` and `args` with a single space, prefixes with `$ `, produces a `'command'` entry
- `createErrorLog(message)` → `{ type: 'error', message: redactSensitiveInfo(message) }`
- `createSuccessLog(message)` → `{ type: 'success', message: redactSensitiveInfo(message) }`

`TaskLogger` (`lib/utils/task-logger.ts`) calls these helpers inside `append` and the convenience methods `info` / `command` / `error` / `success`. The class is instantiated per task via `createTaskLogger(taskId)`, which is called from the route handlers in `app/api/tasks/route.ts`, `app/api/tasks/[taskId]/route.ts`, `app/api/tasks/[taskId]/continue/route.ts`, `app/api/tasks/[taskId]/start-sandbox/route.ts`, and `app/api/tasks/[taskId]/restart-dev/route.ts`.

Two properties of `TaskLogger` matter for the security policy:

- **`append` swallows database write failures.** Every `append` is wrapped in a `try/catch` whose `catch` is empty. A logger failure never propagates to the calling route, so the user-visible logs can be momentarily missing without breaking the task itself. The trade-off is that a broken `redactSensitiveInfo` (for example, a regex compilation error) would also be silently absorbed.
- **`updateProgress` and `updateStatus` re-read the existing `logs` array.** The two mutators do a `SELECT … FROM tasks WHERE id = taskId` to fetch the current `logs`, append the new entry, and persist the merged array inside a single `UPDATE`. Concurrent appends therefore race only on the last entry; a logger that interleaves two writes may lose one. This is a known property of the per-row jsonb append pattern, not a redaction concern.

## Adding or modifying secrets

When a new credential format or environment variable is added, three things must stay in sync:

1. **Catalog it.** Add a row to the *Master catalog* table in [Environment Variables & Secrets](./environment-variables.md), including the primary reader(s) and whether it is required or optional.
2. **Wire a redaction pattern** in `lib/utils/logging.ts`. The catch-all pass already covers most credentials whose names end in `KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `TEAM_ID`, or `PROJECT_ID`. New entries are needed only for unusual names or for prefix-based identifiers such as `SANDBOX_VERCEL_*`. Patterns are applied in order - more specific patterns must precede more general ones.
3. **Update the AGENTS.md "never expose" list.** The policy document is the canonical reference that the grep-based compliance check and the human review both run against.

If the new variable is required for a sandbox-creation flow, also extend `validateEnvironmentVariables` in `lib/sandbox/config.ts` so a misconfigured deploy fails at the gate rather than at the Vercel API call, and add it to the `env=` parameter of `VERCEL_DEPLOY_URL` in `lib/constants.ts` so the "Deploy with Vercel" button never produces a half-configured instance.
