---
type: concept
title: Encryption & Log Redaction
description: The two cryptographic primitives that protect secrets at rest and in transit (AES-256-CBC for database-stored credentials, JWE A256GCM for the session cookie) and the redaction layer that scrubs known credential patterns from any log line that reaches the UI.
tags: [encryption, crypto, jwe, aes, aes-256-cbc, jose, redact, secrets, security, credentials, redaction, logging, jwe-secret, encryption-key]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-c5e751251b730ee67d7f9fb1
    resource: repo://lib/crypto.ts
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Encryption & Log Redaction

The codebase protects secrets with two distinct cryptographic primitives and one text-level scrubbing layer. This page describes those three pieces and the invariants between them. The session cookie and the database-stored secrets use different algorithms and different keys by design, and the redaction function exists as a last line of defense that sits in front of every log line that could reach the browser.

For the cookie handshake and the OAuth token storage that depend on these primitives, see [Authentication & Sessions](./auth-and-sessions.md). For the environment-variable surface that supplies the keys, see [Environment Variables](../operations/environment-variables.md) and [Security & Redaction](../operations/security-and-redaction.md).

## At-rest encryption: AES-256-CBC (`lib/crypto.ts`)

The first primitive encrypts any value that needs to be persisted in Postgres: OAuth access and refresh tokens, user-supplied API keys, and per-connector OAuth client secrets and environment-variable blobs. The implementation lives in `lib/crypto.ts` and is exported as two functions, `encrypt(plaintext: string)` and `decrypt(ciphertext: string)`.

- **Algorithm.** AES-256-CBC, with a fresh 16-byte random IV drawn from `crypto.randomBytes(IV_LENGTH)` on every `encrypt` call. The IV is prepended to the ciphertext in the wire format `<iv_hex>:<ciphertext_hex>`, so each record carries the IV it was encrypted with and the database never needs a separate IV column.
- **Key.** A 32-byte key is derived from `process.env.ENCRYPTION_KEY`, which must be supplied as a 64-character hex string (a literal 32 raw bytes is not accepted). The `getEncryptionKey()` helper parses the hex, enforces the 32-byte length, and returns `null` if the variable is unset, in which case both `encrypt` and `decrypt` throw with the `openssl rand -hex 32` hint from the error message.
- **Wire format.** `iv_hex` and `encrypted_hex` are concatenated with a single colon. `decrypt` splits on the first colon, decodes both halves as hex, runs `crypto.createDecipheriv(ALGORITHM, key, iv)`, and returns the UTF-8 plaintext.
- **Empty string passthrough.** Both functions return the input unchanged when it is an empty string, so callers can pass `connector.oauthClientSecret || ''` and similar guards without a separate check.
- **Failure mode.** `decrypt` rejects any input that does not contain `:` with `'Invalid encrypted text format'`. A native decipher failure is wrapped as `'Failed to decrypt: <message>'`. There is no silent degradation: a corrupt ciphertext throws.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    subgraph Write["Write path"]
        P["plaintext<br/>OAuth token / API key / connector secret"]
        E["encrypt()"]
        K1["ENCRYPTION_KEY<br/>32-byte hex"]
        R1["crypto.randomBytes(16)<br/>per-call IV"]
        S["<iv_hex>:<ciphertext_hex>"]
    end
    subgraph Read["Read path"]
        S2["<iv_hex>:<ciphertext_hex>"]
        D["decrypt()"]
        K2["ENCRYPTION_KEY<br/>32-byte hex"]
        P2["plaintext"]
    end
    P --> E
    K1 --> E
    R1 --> E
    E --> S
    S --> S2
    S2 --> D
    K2 --> D
    D --> P2
```

*Diagram: the AES-256-CBC primitive in `lib/crypto.ts`. The IV is fresh on every encrypt and travels with the ciphertext in the database column; the key is shared across all callers and never persisted.*

### Where the AES-256-CBC primitive is used

The encrypt/decrypt pair is the only at-rest cipher in the codebase. The storage surfaces are:

| Storage surface | Encrypted columns | Encrypted by | Decrypted by |
| --- | --- | --- | --- |
| `users.accessToken`, `users.refreshToken` | Primary OAuth tokens (Vercel + GitHub-primary) | `lib/session/create.ts`, `lib/session/create-github.ts` | `lib/session/get-oauth-token.ts` |
| `accounts.accessToken`, `accounts.refreshToken` | Linked GitHub OAuth tokens | `app/api/auth/github/callback/route.ts` (connect path) | `lib/session/get-oauth-token.ts`, `lib/github/user-token.ts` |
| `keys.value` | Per-user API keys (OpenAI, Gemini, Cursor, Anthropic, AI Gateway) | `app/api/api-keys/route.ts` | `app/api/api-keys/route.ts`, `lib/api-keys/user-keys.ts` |
| `connectors.oauthClientSecret`, `connectors.env` | Per-connector OAuth client secret and env JSON blob | `lib/actions/connectors.ts` (create + update) | `lib/actions/connectors.ts`, `app/api/connectors/route.ts`, `app/api/tasks/route.ts`, `app/api/tasks/[taskId]/continue/route.ts` |

Two reader patterns show up consistently:

- **`getUserApiKeys()` / `getUserApiKey(provider)` in `lib/api-keys/user-keys.ts`** select every row in `keys` for the current user, `decrypt` each `key.value`, and overlay the result onto a base object initialized from `process.env.*`. If no user row is present (or the decryption throws), the function falls back to the system env vars.
- **`getOAuthToken(userId, provider)` in `lib/session/get-oauth-token.ts`** is the canonical OAuth token reader. For GitHub it checks `accounts` first and falls back to `users`; for Vercel it checks only `users`. It returns `{ accessToken, refreshToken: string | null, expiresAt: Date | null }` with `expiresAt` populated only from the `accounts` row, since the primary `users` table does not carry one.
- **`getUserGitHubToken(req?)` in `lib/github/user-token.ts`** is a thin GitHub-only specialization of the same logic and is used by the task worker to inject the user's GitHub token into the sandbox.

The task routes in `app/api/tasks/route.ts` and `app/api/tasks/[taskId]/continue/route.ts` decrypt `connectors.oauthClientSecret` and `connectors.env` (the latter through `JSON.parse(decrypt(connector.env))`) when building the `mcpServers` payload that the agent dispatcher passes into the sandbox. Decryption failures in that block are caught, the task is logged with `Warning: Could not fetch MCP servers, continuing without them`, and the dispatcher is invoked with an empty `mcpServers` array — no retry, no surface error.

## Session cookie encryption: JWE A256GCM (`lib/jwe/`)

The second primitive encrypts the session payload that lives in the `_user_session_` cookie. It is built on the `jose` library and is split into two modules in `lib/jwe/`:

- **`lib/jwe/encrypt.ts`** exports `encryptJWE(payload, expirationTime, secret?)`. It uses `new EncryptJWT(payload).setExpirationTime(expirationTime).setProtectedHeader({ alg: 'dir', enc: 'A256GCM' }).encrypt(base64url.decode(secret))`. The `alg: 'dir'` header means the symmetric key is supplied directly rather than wrapped. The function throws `'Missing JWE secret'` when `process.env.JWE_SECRET` is unset.
- **`lib/jwe/decrypt.ts`** exports `decryptJWE<T>(ciphertext, secret?)`. It uses `jwtDecrypt` with `base64url.decode(secret)`, returns the typed payload (defaulting `T` to `string | object`), and deletes the `iat` and `exp` claims from any object payload before returning so the caller does not have to. **It silently swallows any error from `jwtDecrypt`** and returns `undefined`, which is the entire reason a tampered, expired, or otherwise unreadable cookie degrades to "not signed in" rather than a 500. The function also throws `'Missing JWE secret'` if the secret is unset.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: a semicolon inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    subgraph Sign["Sign-in: createSession / createGitHubSession"]
        Sess["Session object<br/>{ created, authProvider, user }"]
        Enc["encryptJWE(s, '1y')"]
        K1["JWE_SECRET<br/>base64url"]
        JW["JWE compact<br/>alg=dir, enc=A256GCM<br/>exp=1y"]
    end
    subgraph Read["Every request: getSessionFromCookie"]
        C["Cookie value"]
        Dec["decryptJWE&lt;Session&gt;"]
        K2["JWE_SECRET<br/>base64url"]
        S2["Session object<br/>or undefined"]
    end
    Sess --> Enc
    K1 --> Enc
    Enc --> JW
    JW --> C
    C --> Dec
    K2 --> Dec
    Dec -- "valid" --> S2
    Dec -- "expired / tampered" --> S2
```

*Diagram: the JWE primitive in `lib/jwe/`. The same `JWE_SECRET` key is used on both sides; decryption never throws, so a malformed cookie is indistinguishable from no cookie at all.*

The two endpoints are:

- **Write side.** Both `saveSession` implementations (`lib/session/create.ts` for Vercel, `lib/session/create-github.ts` for GitHub) call `encryptJWE(session, '1y')`, append a `Set-Cookie` header with `Path=/`, `HttpOnly`, `SameSite=Lax`, `Max-Age=COOKIE_TTL` (also `'1y'` via the `ms` package), and `Secure` only when `NODE_ENV === 'production'`. Passing `undefined` to `saveSession` produces an immediate-expiry `Set-Cookie`, which is how `/api/auth/signout` clears the cookie.
- **Read side.** `getSessionFromCookie(cookieValue?)` in `lib/session/server.ts` is the only function that actually decrypts the cookie, returning a `Session` with the `iat`/`exp` claims already stripped. `getServerSession` (Next.js server-component path, `cookies()`) and `getSessionFromReq` (route-handler path, `NextRequest`) both delegate to it.

The `'dir'` + `A256GCM` algorithm pair plus the absence of a server-side session table means the cookie **is** the session: rotating `JWE_SECRET` invalidates every existing session immediately, and any process that holds `JWE_SECRET` can both forge and read every cookie. This is documented in the auth page and is the reason `JWE_SECRET` and `ENCRYPTION_KEY` are listed side by side in AGENTS.md under "Never expose these in logs".

## Log redaction: `redactSensitiveInfo` (`lib/utils/logging.ts`)

The third piece is text-level. `redactSensitiveInfo(message: string): string` in `lib/utils/logging.ts` is the function that scrubs any log string before it is written to the per-task `logs` JSON column (which the UI streams back to the browser) or any other user-visible destination. It is invoked from exactly one place: `createLogEntry(type, message, timestamp?)`, which is the helper that `TaskLogger.append` (`lib/utils/task-logger.ts`) and the convenience methods `info`/`command`/`error`/`success` use for every log line.

### The wrapper chain

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    C["Sandbox command / stdout / stderr"]
    R["redactSensitiveInfo()"]
    CE["createLogEntry()<br/>also calls redactSensitiveInfo"]
    L["TaskLogger<br/>.info / .command / .error / .success"]
    DB["tasks.logs<br/>(jsonb, streamed to UI)"]
    C --> R
    R --> CE
    CE --> L
    L --> DB
```

*Diagram: the path from a sandbox command's text output to the UI-facing `tasks.logs` column. The redact step runs twice for defense in depth — once at the call site, once inside `createLogEntry`.*

Because every log line the UI sees passes through `createLogEntry`, every agent file that writes user-visible logs calls `redactSensitiveInfo` again at the source. The agent files that do this are:

- `lib/sandbox/creation.ts` — wraps `runCommandInSandbox` in a `runAndLogCommand` helper that redacts the command, stdout, and stderr before `logger.command`/`logger.info`/`logger.error`.
- `lib/sandbox/agents/claude.ts`, `codex.ts`, `copilot.ts`, `cursor.ts`, `gemini.ts`, `opencode.ts` — each one has its own `runAndLogCommand` helper that runs `redactSensitiveInfo` over the same three strings.

`opencode.ts` also strips inline `API_KEY="…"` literals from the command line via a literal `replace(/API_KEY="[^"]*"/g, 'API_KEY="[REDACTED]"')` step before the broader `redactSensitiveInfo` pass, because the OpenCode launch command inlines several `API_KEY="…"` pairs as environment prefixes.

### What `redactSensitiveInfo` matches

The function runs three passes over the input, in this order:

1. **API key patterns.** A list of regular expressions targeting common provider credential shapes. Each match preserves the leading variable name and the first four plus last four characters of the value, with the middle replaced by `'*'` (or fully replaced with `'*'` if the value is eight characters or shorter). The exact category coverage is:
   - `ANTHROPIC_API_KEY` followed by an `sk-ant-…` value.
   - `OPENAI_API_KEY` followed by an `sk-…` value.
   - `GITHUB_TOKEN` followed by a `ghp_/gho_/ghu_/ghs_/ghr_…` value.
   - GitHub auth-in-URL form: `https://<token>(:x-oauth-basic)?@github.com`. This pattern is special-cased to preserve the URL structure (the host part is kept) while replacing only the token segment with the first-4/last-4 mask.
   - Generic `API_KEY=<value>` (an `API_KEY` name followed by a long alphanumeric value).
   - `Bearer <value>` headers.
   - Generic `TOKEN=<value>`.
   - `SANDBOX_VERCEL_TEAM_ID`, `SANDBOX_VERCEL_PROJECT_ID`, and `SANDBOX_VERCEL_TOKEN` followed by their respective alphanumeric values.
2. **JSON field pattern.** `"teamId"` and `"projectId"` keys (matched case-insensitively, with optional whitespace and a colon) have their string values replaced with `"[REDACTED]"`. This is the rule that catches a Vercel sandbox config blob such as `{ teamId: "team_abc123", projectId: "prj_xyz" }` and turns it into `{ teamId: "[REDACTED]", projectId: "[REDACTED]" }`.
3. **Environment variable assignments.** A broader catch-all that matches any uppercase variable name ending in `KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `TEAM_ID`, or `PROJECT_ID` (with optional leading/trailing underscores) followed by an alphanumeric value of at least 8 characters. This is what scrubs the inline `OPENAI_API_KEY=sk-ant-***` strings an agent CLI may echo back.

The output of every pass is a string with the variable name preserved and the credential value masked. Pass 1 runs first because its patterns are more specific; pass 3 catches everything pass 1 missed.

### Redaction is a backup, not a defense

Per `AGENTS.md`, the project's primary defense against leaking credentials into user-visible logs is the rule that `logger.*` and `console.*` calls in user-facing paths use static strings only. `redactSensitiveInfo` exists because real sandbox output contains dynamic values (command lines, agent stderr, package-manager logs) that bypass the static-string rule by construction. The masking guarantees that if a credential appears in that stream, only its first four and last four characters reach the UI. The function deliberately does not redact any of the variable names themselves — only the values — so the UI can still tell *which* key leaked.

## Log-entry helpers built on the redaction layer

`lib/utils/logging.ts` exports four thin convenience helpers on top of `createLogEntry`, all of which feed into the `TaskLogger` (`lib/utils/task-logger.ts`) and therefore into the `tasks.logs` jsonb column streamed to the UI:

- `createInfoLog(message)` → `{ type: 'info', message: redactSensitiveInfo(message), timestamp }`.
- `createCommandLog(command, args?)` → joins `command` and `args` with a single space, prefixes with `$ `, and produces a `'command'` entry.
- `createErrorLog(message)` → `{ type: 'error', message: redactSensitiveInfo(message) }`.
- `createSuccessLog(message)` → `{ type: 'success', message: redactSensitiveInfo(message) }`.

`TaskLogger.append` is wrapped in a try/catch that **silently absorbs database write failures**, so a logger failure cannot break the surrounding task. `TaskLogger.updateProgress` and `TaskLogger.updateStatus` re-read the existing `logs` array from `tasks`, append a new entry, and persist it inside a single update so concurrent appends can race only on the last entry.

## Configuration surface

Both cryptographic primitives read their keys from environment variables and refuse to operate without them:

| Variable | Used by | Format | Failure when missing |
| --- | --- | --- | --- |
| `ENCRYPTION_KEY` | `lib/crypto.ts` (`encrypt`/`decrypt`) | 32-byte hex string (64 characters) | Throws with the `openssl rand -hex 32` hint |
| `JWE_SECRET` | `lib/jwe/encrypt.ts`, `lib/jwe/decrypt.ts` | Base64url-encoded (per `README.md`, `openssl rand -base64 32`) | `encryptJWE` and `decryptJWE` throw `'Missing JWE secret'`; `decryptJWE` then degrades to `undefined` on a missing secret only when the failure is the `getServerSession` read path that already handles `undefined` |

The two keys are deliberately separate so rotating one does not invalidate the other. Rotating `JWE_SECRET` logs every user out instantly (their cookies can no longer be decrypted). Rotating `ENCRYPTION_KEY` requires every user to re-run the OAuth flow and re-enter every API key, because every ciphertext stored in Postgres was bound to the previous key.

`redactSensitiveInfo` has no configuration surface — its patterns are compiled into the function — so adding a new credential format requires editing `lib/utils/logging.ts`. There is no allowlist, no environment toggle, and no per-call option to disable redaction.

## Failure and edge-case behavior

- **`ENCRYPTION_KEY` missing or wrong length.** `getEncryptionKey()` returns `null` or throws; `encrypt` and `decrypt` then throw, and any database row that was previously encrypted becomes unreadable. The task routes catch this only in the connector-decryption block (`Warning: Could not fetch MCP servers, continuing without them`); everywhere else the throw propagates and the route returns 500.
- **`JWE_SECRET` missing.** Every sign-in or sign-out call throws before it can write the cookie. Every request whose `getServerSession` / `getSessionFromReq` path calls `decryptJWE` returns `undefined`, which all auth-gated routes treat as 401.
- **Malformed ciphertext (`crypto.ts`).** `decrypt` throws `'Invalid encrypted text format'` if the input lacks a colon; callers like `/api/vercel/teams` return 401 and log the error.
- **Expired or tampered JWE (`lib/jwe/decrypt.ts`).** `jwtDecrypt` throws inside the `try`, the `catch` block swallows the error, and the function returns `undefined`. The session is treated as absent. There is no server-side audit log of this event.
- **Redaction not a 100% guarantee.** `redactSensitiveInfo` is a regex pass; values that do not match one of the listed patterns will reach the UI verbatim. The `AGENTS.md` rule that every user-facing log call must use static strings is the only way to keep that window closed; the redaction layer only narrows it.

## Extension points

- **Adding a new at-rest secret.** Add an `encrypt(value)` at the write site and a `decrypt(column)` at the read site, using the existing `ENCRYPTION_KEY`. No new key, no new module, no new table column for the IV (the IV rides inside the ciphertext). If the new secret format has a recognizable prefix or value shape, also add a pattern to `redactSensitiveInfo` so it cannot leak through agent stdout.
- **Adding a new credential format to the redaction layer.** Edit the `apiKeyPatterns` array in `lib/utils/logging.ts`. Patterns are applied in order; more specific patterns must precede more general ones.
- **Switching the at-rest cipher.** Replacing AES-256-CBC would require a migration of every encrypted column, plus an update to the wire format expected by `decrypt`. AES-256-CBC is the only choice the codebase makes here; the `ALGORITHM` constant is the single point of change.
- **Switching the cookie cipher.** The JWE helpers are isolated to `lib/jwe/`, but changing the algorithm pair (`alg`/`enc`) or the secret shape (e.g. moving from base64url to hex) would invalidate every existing session cookie. The `alg: 'dir', enc: 'A256GCM'` pair is the only one currently supported.
