---
type: testing
title: Testing Strategy & Coverage
description: The project's current testing posture (no formal unit/integration tests; verification rests on pnpm type-check, pnpm lint, pnpm format, and the pnpm build), and a prioritized backlog of unit and integration tests per subsystem.
tags: [testing, test-strategy, coverage, type-check, lint, build, vitest, unit-test, integration-test, compliance, security, redaction, encryption, jwe, sandbox, auth, upsert-user, rate-limit]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-6d4b4e707b8d60b6ccfa3425
    resource: repo://.github/workflows/openwiki-update.yml
  - id: openwiki-source-8037e2358a2c4f9b2c722a11
    resource: repo://AGENTS.md
  - id: openwiki-source-c5e751251b730ee67d7f9fb1
    resource: repo://lib/crypto.ts
  - id: openwiki-source-2895634f2776df5a6886c6eb
    resource: repo://lib/db/users.ts
  - id: openwiki-source-8b71134f021e5bd57558cf48
    resource: repo://lib/jwe/decrypt.ts
  - id: openwiki-source-2972832ca9085b9e225f8b49
    resource: repo://lib/jwe/encrypt.ts
  - id: openwiki-source-78237118aac1cf9686d70d4d
    resource: repo://lib/sandbox/config.ts
  - id: openwiki-source-b4de2cd9d50e247e61519a30
    resource: repo://lib/utils/branch-name-generator.ts
  - id: openwiki-source-4c0dfa7b4928caf4818e73f8
    resource: repo://lib/utils/logging.ts
  - id: openwiki-source-7ece006b3c5e6a3e70e9b390
    resource: repo://lib/utils/rate-limit.ts
  - id: openwiki-source-5b54a58d1b51cd490b0e7162
    resource: repo://package.json
  - id: openwiki-source-1bc696df8ff4275baeff50ac
    resource: repo://scripts/migrate-production.ts
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Testing Strategy & Coverage

The repository does **not** ship a formal unit or integration test suite. There is no test runner configured (`vitest`, `jest`, or similar), no `test` script in `package.json`, no `*.test.ts` / `*.spec.ts` files, and no `__tests__/` directory. The "tests" the project actually runs are the static-analysis pipeline plus a small set of `grep`-based compliance checks in `AGENTS.md`. This page describes that reality, inventories what each gate catches, and recommends the high-value unit and integration tests that would most improve safety if a runner were added.

For the cryptographic primitives that are the most obvious target for unit tests, see [Encryption & Log Redaction](../concepts/encryption-and-redaction.md). For the per-subsystem call graphs a future integration suite would exercise, see [Sandbox Orchestration](../systems/sandbox-orchestration.md) and [Auth API Surface](../systems/auth-flow.md). For the schema-change pipeline that runs only in production, see [Database Migrations](../operations/database-migrations.md).

## Current posture: static checks, not tests

`package.json` defines seven project commands and four `db:*` commands; none of them invoke a test runner:

```json
"scripts": {
  "dev": "next dev --webpack",
  "build": "next build --turbopack",
  "start": "next start",
  "lint": "eslint",
  "type-check": "tsc --noEmit",
  "format": "prettier --write \"**/*.{ts,tsx}\"",
  "format:check": "prettier --check \"**/*.{ts,tsx}\"",
  "db:generate": "drizzle-kit generate",
  "db:migrate": "drizzle-kit migrate",
  "db:push": "drizzle-kit push",
  "db:studio": "drizzle-kit studio",
  "prepare": "husky"
}
```

No `test`, `test:unit`, `test:integration`, or `test:watch` script exists. The dev-dependency list (`eslint`, `prettier`, `typescript`, `tsx`, `husky`, `dotenv`, plus the `@types/*` packages and Tailwind toolchain) contains no test runner, no `jsdom`, no `@testing-library/*`, and no `nock`/`msw`. A repo-wide search for `*.test.ts`, `*.spec.ts`, `__tests__/`, `vitest.config.*`, and `jest.config.*` returns zero matches.

`AGENTS.md` *What to Do Instead* references `pnpm test` only as a hypothetical: *"Use `pnpm test` if tests are available"*. The Compliance Checklist at the bottom of the same file codifies the gates that substitute for tests; the four core scripts (`pnpm format`, `pnpm type-check`, `pnpm lint`, `pnpm build`) are the verification surface today.

## What runs today

Five commands form the verification pipeline. They run in this order and catch different classes of bug:

| Command | Tool | What it catches | What it does *not* catch |
| --- | --- | --- | --- |
| `pnpm type-check` | `tsc --noEmit` | Type mismatches, missing imports, wrong arity, broken inference across modules, invalid `as` casts, non-null assertion drift | Any runtime-only bug that the types permit (e.g., wrong branch logic, off-by-one in a date window) |
| `pnpm lint` | `eslint` via `eslint-config-next` | React / Next.js rule violations, unused imports, accessibility regressions, common foot-guns flagged by the Next preset | Logic errors, regressions in business rules, security policy violations that no rule knows about |
| `pnpm format:check` | `prettier --check` | Formatting drift (semi, singleQuote, printWidth 120, trailingComma all from the `prettier` block in `package.json`) | Anything semantic; format failures block CI but do not indicate a real defect |
| `pnpm build` | `next build --turbopack` | Compile-time and RSC errors, missing env vars surfaced by `next build`, broken dynamic imports, broken server/client boundary | Anything that only manifests at runtime (the build does not execute handlers) |
| `pnpm db:generate` | `drizzle-kit generate` | Drift between `lib/db/schema.ts` and the committed `lib/db/migrations/*.sql` | Anything not represented in the schema diff (the migration logic, the order of operations) |

`AGENTS.md` *Testing Changes* adds a sixth gate that is not a script but a grep:

```bash
# Check for logger statements with template literals
grep -r "logger\.(info|error|success|command)\(\`.*\$\{" .

# Check for console statements with template literals
grep -r "console\.(log|error|warn|info)\(\`.*\$\{" .
```

These two commands are the project's only security-policy regression test: any match is a violation of the *CRITICAL: No Dynamic Values in Logs* rule and a failed compliance check. See [Security & Log Redaction Rules](../operations/security-and-redaction.md) for the rule itself and the secrets list it protects.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    subgraph Local["Pre-submission verification"]
      A["pnpm format"] --> B["pnpm type-check"]
      B --> C["pnpm lint"]
      C --> D["pnpm build"]
      E["grep template literals<br/>logger.* / console.*"] --> C
    end
    subgraph Schema["Schema change gate"]
      F["edit lib/db/schema.ts"] --> G["pnpm db:generate"]
      G --> H["review new SQL in lib/db/migrations"]
      H --> I["pnpm db:migrate against local DB"]
    end
    Local -- "edit code" --> F
    I -- "commit migration" --> Local
```

*Diagram: the verification pipeline as it exists today. The middle column is static analysis; the rightmost column is the schema-change loop. Both run on developer machines; neither is wired into CI.*

## CI status

There is exactly one workflow file in `.github/workflows/`: `openwiki-update.yml`, which runs the OpenWiki docs generator on a daily cron and on manual dispatch. It does **not** invoke `pnpm type-check`, `pnpm lint`, `pnpm build`, or any test runner. There is no `pull_request` trigger, no `push` trigger, and no `ci.yml` / `test.yml` workflow. The OpenWiki PR-creation step runs `openwiki code --update --print` with `continue-on-error: true`; a doc-generator failure does not block the deployment.

The implication is that the verification pipeline runs only on the developer's machine and only when the developer chooses to run it. There is no enforced gate that catches a regression before it reaches production.

## High-value unit tests per subsystem

The following list is a prioritized backlog of the unit tests that would most improve safety if a runner (`vitest` is the natural choice given the project's toolchain) were added. Each entry cites the source symbol and the invariants a test should pin down. The list is deliberately short - the goal is high signal, not exhaustive coverage.

### `redactSensitiveInfo` in `lib/utils/logging.ts`

This is the single most important unit-test target in the codebase. The function is security-critical (a missed pattern leaks credentials to the UI), it is a pure function over a string, and the regex pass order matters. A small focused suite would lock in the contract that the static-string rule in `AGENTS.md` is the primary defense and `redactSensitiveInfo` is the secondary defense.

- Returns the input unchanged when no sensitive pattern is present.
- Masks an `ANTHROPIC_API_KEY=sk-ant-<…>` value to first-4 + stars + last-4 while preserving the variable name.
- Handles the GitHub URL form `https://<token>(:x-oauth-basic)?@github.com` and keeps the URL shape intact while masking the embedded token.
- Replaces `"teamId"` / `"projectId"` JSON field values with `"[REDACTED]"`.
- Catches the env-var catch-all suffix set (`KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `TEAM_ID`, `PROJECT_ID`) for variable names that the first pass missed.
- Confirms the known gap: `POSTGRES_URL` is *not* in the regex set, so a string containing `postgres://user:pass@host/db` passes through unmasked - which is exactly the gap the static-string rule exists to close.
- Verifies that values of length ≤ 8 are fully starred rather than first-4 / last-4.

### `validateEnvironmentVariables` in `lib/sandbox/config.ts`

Pure function, table-driven inputs, central to the sandbox-creation gate. A unit test should:

- Return `valid: true` and an empty `error` when every required variable is supplied via `process.env` or the `apiKeys` argument.
- Return one specific error per agent (`claude`, `cursor`, `codex`, `gemini`) when the corresponding key is missing.
- Return the *Either AI_GATEWAY_API_KEY or ANTHROPIC_API_KEY is required* error for `opencode` when neither is set, and `valid: true` when at least one is present.
- Always require a non-null `githubToken` and the three `SANDBOX_VERCEL_*` variables, regardless of agent.
- Default `selectedAgent` to `claude` and the `apiKeys` object to `undefined` (so a missing third argument does not crash).

### `encrypt` / `decrypt` round-trip in `lib/crypto.ts`

AES-256-CBC is a deterministic primitive that only fails when inputs are malformed. The tests should pin down the wire format and the failure modes:

- `decrypt(encrypt(x)) === x` for any non-empty UTF-8 string, with the IV regenerated each call so two encryptions of the same plaintext produce different ciphertexts.
- `encrypt('')` and `decrypt('')` return the empty string unchanged.
- `encrypt` throws when `ENCRYPTION_KEY` is unset.
- `encrypt` throws when `ENCRYPTION_KEY` is the wrong length (not 64 hex characters).
- `decrypt` throws `'Invalid encrypted text format'` when the input has no `:`.
- `decrypt` throws `'Failed to decrypt: …'` when the input is well-formed but the key is wrong.

### `encryptJWE` / `decryptJWE` round-trip in `lib/jwe/`

The session-cookie primitive. Two complementary tests:

- `decryptJWE<T>(await encryptJWE<T>(payload, '1y', secret))` returns the original payload (string or object) for both `string` and `object` payloads.
- `decryptJWE` returns `undefined` (does **not** throw) for tampered ciphertext, an expired `expirationTime`, or a different secret - this is the property that lets a bad cookie degrade to "not signed in" instead of a 500.
- Both functions throw `'Missing JWE secret'` when `JWE_SECRET` is unset.
- Object payloads returned by `decryptJWE` have `iat` and `exp` stripped.

### `upsertUser` dedup in `lib/db/users.ts`

The user-creation function has three branches that all need to resolve to "one canonical user":

- If a primary user row exists for `(provider, externalId)`, update tokens and last-login, return the existing `id`.
- If the provider is GitHub and the external ID is already linked via the `accounts` table (the "signed in with Vercel, later signed in with GitHub" path), reuse the existing user.
- Otherwise insert a new row with a fresh `nanoid` and return the new `id`.

A unit test needs a Drizzle mock or a per-test Postgres schema (the `drizzle-kit push --force` flow already in `package.json` makes the latter cheap). The three branches above plus the "no duplicate user on second sign-in" invariant are the four tests to add.

### `checkRateLimit` in `lib/utils/rate-limit.ts`

A DB-backed function but the date-window logic is worth pinning down:

- The window is `[today UTC midnight, tomorrow UTC midnight)`; tasks created at 23:59 UTC do not count against the next day.
- Soft-deleted tasks (`isNull(tasks.deletedAt)`) are excluded from both `tasksToday` and `userMessagesToday`.
- `remaining` is clamped to `Math.max(0, maxMessagesPerDay - count)` so the field never reports negative.
- `allowed === false` once `count >= maxMessagesPerDay`.
- The total combines new tasks and user messages, not just one of the two.

### `createFallbackBranchName` in `lib/utils/branch-name-generator.ts`

Pure and deterministic, ideal for a snapshot-style unit test:

- Output starts with `agent/` and ends with the first 8 characters of `taskId`.
- The timestamp segment is derived from `new Date()` and contains no colons or dots.
- `createFallbackBranchName('abc12345-xyz')` for two different `Date` values produces different timestamps.

The companion `generateBranchName` function requires a mocked `generateText` from `ai` and would need a fixture for the prompt to be stable. Lower priority than the fallback because the failure mode (an invalid branch name) is loud rather than silent.

### `createAuthenticatedRepoUrl` and `createSandboxConfiguration` in `lib/sandbox/config.ts`

Two pure builders with no I/O. The first should be tested with:

- A GitHub URL with a token: `username` set to the token, `password` set to `x-oauth-basic`, the rest of the URL unchanged.
- A non-GitHub URL with a token: returned unchanged (no credentials injected).
- A GitHub URL without a token: returned unchanged.
- A malformed URL string: returned unchanged (the `catch` block).

`createSandboxConfiguration` should be tested for its defaults (`template: 'node'`, `branch: 'main'`, `timeout: '20m'`, `ports: [3000]`, `runtime: 'node22'`, `resources: { vcpus: 4 }`) and for override behavior when each field is supplied.

### URL and number helpers

Pure, deterministic, low-risk - ideal for a starter test suite:

- `isRelativeUrl` (`lib/utils/is-relative-url.ts`): returns `true` for `'foo/bar'`, `'./x'`, `'../x'`; returns `false` for `'https://example.com/x'`.
- `formatAbbreviatedNumber` (`lib/utils/format-number.ts`): `999 → '999'`, `1000 → '1k'`, `1500 → '1.5k'`, `999999 → '999.9k'` (boundary), `1000000 → '1M'`, `1100000 → '1.1M'`.
- `getEnabledAuthProviders` (`lib/auth/providers.ts`): `NEXT_PUBLIC_AUTH_PROVIDERS` unset returns `{ github: true, vercel: false }`; set to `'github,vercel'` returns both `true`; set to `'google'` returns the default (unknown providers are silently ignored).
- `generateId` (`lib/utils/id.ts`): produces strings of the requested length using the `nanoid` alphabet.

## Integration tests that would matter most

Integration tests need a runner with a database fixture and an HTTP harness (Vitest + `vite-node`, or Playwright for the UI). Three integration suites would catch the regressions that no unit test can:

### Auth callback → user row → session

Mock the OAuth provider and the database. Drive `app/api/auth/callback/vercel/route.ts` and `app/api/auth/github/callback/route.ts` through:

- A successful first-time sign-in: creates a `users` row, returns a JWE cookie, the cookie decrypts to the session payload.
- A sign-in with a GitHub external ID that is already linked via the `accounts` table (the Vercel-then-GitHub path): reuses the existing user, does **not** create a duplicate row.
- A callback with mismatched or missing state: returns 400, deletes the cookies only on success.
- A `signout/route.ts` call: clears the cookie and revokes the upstream token (the revoke call is mocked).

### Sandbox config → env gate → SDK call

Mock `@vercel/sandbox`. Drive `createSandbox` through:

- A request with all `SANDBOX_VERCEL_*` variables set: `validateEnvironmentVariables` returns `{ valid: true }`, `Sandbox.create` is invoked with the URL from `createAuthenticatedRepoUrl`.
- A request missing one of the required variables: the handler short-circuits with the same `error` string the validator returns and never calls `Sandbox.create`.
- A request for a Vite repo: `detectPortFromRepo` returns `5173`; otherwise `3000`.

### Migration round-trip

Against a throwaway Postgres schema (Neon branch or local Docker):

- `pnpm db:generate` after a schema edit produces a migration whose SQL, applied via `pnpm db:migrate`, brings a fresh schema to the same state as the edited `lib/db/schema.ts`.
- `pnpm db:push --force` on an existing database wipes and rebuilds it; `pnpm db:migrate` on the same schema is a no-op (the journal catches up).
- `scripts/migrate-production.ts` with `VERCEL_ENV='production'` and `POSTGRES_URL` set invokes `drizzle-kit migrate` and exits 0; with `VERCEL_ENV='preview'` exits 0 without touching the database; with `VERCEL_ENV='production'` and `POSTGRES_URL` unset throws.

## What the static checks already cover (and why they are not enough)

`pnpm type-check` is the strongest gate. The codebase uses `zod` for runtime validation at boundaries (`lib/db/schema.ts` derives the row types), so most state transitions are typed end-to-end. A change that breaks the type system is caught before review. But the type system cannot catch:

- A regex in `redactSensitiveInfo` that no longer matches a real provider key shape (e.g., if Anthropic rotates the `sk-ant-` prefix).
- A date-window bug in `checkRateLimit` where the UTC boundary moves the wrong way at midnight.
- An `upsertUser` branch that creates two user rows for the same external ID.
- An `isRelativeUrl` implementation that returns `true` for an absolute URL because `new URL` was passed the wrong argument.

These are the four classes of bug a focused unit test suite would catch deterministically. They are also the four classes of bug that would most directly degrade a user-facing property (security, rate-limit correctness, account uniqueness, redirect safety).

## Extending the verification pipeline

If a runner is added, three low-cost changes would make the existing scripts stricter without adding any new tool:

- Wire `pnpm type-check`, `pnpm lint`, `pnpm build`, and (once added) `pnpm test` into a `.github/workflows/ci.yml` triggered on `pull_request` and `push` to `main`. Today, only the OpenWiki workflow runs in GitHub Actions.
- Add the AGENTS.md grep commands as a separate `lint:logs` script so the static-string rule can run in CI the same way it runs locally:
  ```json
  "lint:logs": "node -e \"const r=require('child_process').execSync; try{r(\\\"grep -rE 'logger\\\\.(info|error|success|command)\\\\\\\\\\\\(`.*\\\\$\\\\{' .\\\",{stdio:'pipe'})}catch(e){process.exit(1)} try{r(\\\"grep -rE 'console\\\\.(log|error|warn|info)\\\\\\\\\\\\(`.*\\\\$\\\\{' .\\\",{stdio:'pipe'})}catch(e){process.exit(1)}\""
  ```
- Add a script that walks `app/**/page.tsx` and `components/**` and fails if any `process.env.*` reference other than `NEXT_PUBLIC_AUTH_PROVIDERS` and `NEXT_PUBLIC_GITHUB_CLIENT_ID` is found - the client-safe boundary from AGENTS.md, currently enforced only in review.

These three additions would close the gap between "the developer remembered to run the script" and "the script is guaranteed to have run" without requiring a test runner. They are the cheapest possible investment in the existing verification posture.

## Summary

The project's verification today is a five-command static-analysis pipeline plus two `grep` checks for the logging rule. There is no test runner, no test files, and no CI workflow that runs any of these scripts on every change. The highest-leverage additions would be a small vitest suite covering `redactSensitiveInfo`, `validateEnvironmentVariables`, the `encrypt`/`decrypt` and `encryptJWE`/`decryptJWE` round-trips, and the `upsertUser` dedup branches; a CI workflow that runs `type-check` + `lint` + `build` (and eventually `test`) on every PR; and the `lint:logs` and client-safe-env scripts above. The other subsystems have integration boundaries that would benefit from fixtures, but the four unit-test targets above are where the existing static checks are weakest and the cost of a regression is highest.
