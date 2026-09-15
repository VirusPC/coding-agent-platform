---
type: concept
title: Authentication & Sessions
description: The two primary OAuth providers (GitHub and Vercel), the JWE-encrypted cookie session, the linking model between Vercel users and connected GitHub accounts, and where OAuth token decryption happens.
tags: [auth, oauth, github, vercel, jwe, session, cookies, encryption, accounts]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-6e7fdb664c7401491a0a2550
    resource: repo://app/api/auth/callback/vercel/route.ts
  - id: openwiki-source-1745d192079a6690c9cd73cc
    resource: repo://app/api/auth/github/callback/route.ts
  - id: openwiki-source-9ae4d3c854e5a4c6ef24eeb3
    resource: repo://app/api/auth/signin/github/route.ts
  - id: openwiki-source-552741c58d746b8418a79f4f
    resource: repo://app/api/auth/signin/vercel/route.ts
  - id: openwiki-source-0dd0bd54c06ca50efe9ecefa
    resource: repo://app/api/auth/signout/route.ts
  - id: openwiki-source-f4a535c4711843f9246eaee1
    resource: repo://lib/auth/providers.ts
  - id: openwiki-source-c5e751251b730ee67d7f9fb1
    resource: repo://lib/crypto.ts
  - id: openwiki-source-526feb2334febba699169a23
    resource: repo://lib/db/schema.ts
  - id: openwiki-source-2895634f2776df5a6886c6eb
    resource: repo://lib/db/users.ts
  - id: openwiki-source-23b6c2f4d167395f7cc43e09
    resource: repo://lib/github/user-token.ts
  - id: openwiki-source-8b71134f021e5bd57558cf48
    resource: repo://lib/jwe/decrypt.ts
  - id: openwiki-source-2972832ca9085b9e225f8b49
    resource: repo://lib/jwe/encrypt.ts
  - id: openwiki-source-41585b3256131ff4461eef45
    resource: repo://lib/session/constants.ts
  - id: openwiki-source-62c88e4f8b7c5177387a7591
    resource: repo://lib/session/create-github.ts
  - id: openwiki-source-726803282aeeefa3d1ff5d90
    resource: repo://lib/session/create.ts
  - id: openwiki-source-abf11954c79e94f843b63430
    resource: repo://lib/session/get-oauth-token.ts
  - id: openwiki-source-053c6c17433a95fb01a35097
    resource: repo://lib/session/get-server-session.ts
  - id: openwiki-source-47782d3e9e3530f046543036
    resource: repo://lib/session/server.ts
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Authentication & Sessions

This page describes how the application identifies users, how their session lives in a browser cookie, and how a single user can attach multiple OAuth identities (a primary sign-in plus an optional linked GitHub account). It is the concept page for the auth subsystem. For per-provider integration details see GitHub OAuth and Vercel OAuth. For the encryption primitives used for both cookie payloads and stored OAuth tokens, see Encryption and Redaction. The step-by-step end-to-end dance is in Authentication Flow and the user-driven connect path is in Connect GitHub.

## Two primary OAuth providers

The application supports exactly two identity providers, selected at deploy time by the `NEXT_PUBLIC_AUTH_PROVIDERS` environment variable (a comma-separated list such as `github`, `vercel`, or `github,vercel`). The default is `github` when the variable is unset.

`getEnabledAuthProviders` in `lib/auth/providers.ts` parses that variable into a `{ github, vercel }` boolean record, which the sign-in UI consumes to decide which buttons to render. Both providers share the same downstream mechanics (a `users` row plus a JWE cookie) but they differ in how they arrive there.

| Provider | OAuth flavor | Sign-in route | Callback route | Profile retrieval |
| --- | --- | --- | --- | --- |
| GitHub | Standard auth code, no PKCE, scopes `repo,read:user,user:email` | `GET /api/auth/signin/github` (plus legacy `GET/POST /api/auth/github/signin`) | `GET /api/auth/github/callback` | `https://api.github.com/user` (and `/user/emails` if private) |
| Vercel | Auth code with PKCE (S256), default scopes | `POST /api/auth/signin/vercel` | `GET /api/auth/callback/vercel` | `https://api.vercel.com/v2/user` with fallback to `https://vercel.com/api/www/user` |

The two sign-in routes are deliberately non-symmetric. Vercel's start route is `POST` because the `arctic` PKCE flow returns the authorize URL to a client that has to navigate to it, while GitHub's start route is `GET` and redirects directly. The `arctic` library supplies `generateState`, `generateCodeVerifier`, `OAuth2Client.createAuthorizationURLWithPKCE`, and `validateAuthorizationCode`. The latter is the only piece that has to be called with Vercel's bespoke token endpoint `https://vercel.com/api/login/oauth/token` because `arctic` does not ship a Vercel adapter.

## The `users` and `accounts` split

Identities are stored in two Drizzle tables defined in `lib/db/schema.ts`:

- **`users`** holds the *primary* OAuth identity, i.e. the one the user actually signed in with. Its `provider` enum is `'github' | 'vercel'`, and the unique index `(provider, externalId)` guarantees one row per `(provider, externalId)` pair. `users.id` is a server-generated `nanoid` and is the canonical internal user identifier referenced by `tasks`, `connectors`, `keys`, `accounts`, and `settings`.
- **`accounts`** holds *additional* linked OAuth identities. Its `provider` enum is currently restricted to `'github'` only, and the unique index `(userId, provider)` enforces "one GitHub link per internal user". The same external GitHub user ID can appear in `accounts` rows belonging to multiple internal users, which is the precondition for the account merge described below.

The asymmetric split (Vercel can be a primary but never a link, GitHub can be both) is enforced in two places:

1. The `accounts.provider` schema enum is hard-coded to `'github'`, so the database itself rejects any other value.
2. The connect flow's two call sites use `provider: 'github'` exclusively when inserting or updating `accounts` rows.

A Vercel user can attach one GitHub identity for repository access. A GitHub-primary user has nowhere to attach a Vercel identity because Vercel does not act as a repository authorization source.

## `upsertUser`: the three-stage dedup

`upsertUser` in `lib/db/users.ts` is the single function that creates or finds the internal `users.id` for a given `(provider, externalId)` pair after a successful OAuth code exchange. It runs three checks in order, and the order matters because each later check is broader:

1. **Same primary exists.** If a `users` row already matches `(provider, externalId)`, treat this as a returning sign-in. Update `accessToken`, `refreshToken`, `scope`, profile fields, `updatedAt`, and `lastLoginAt`, and return the existing `users.id`. This is the steady-state case for any user who signs in with the same provider repeatedly.
2. **GitHub already linked to a different user.** If the provider is `'github'` and the same GitHub numeric ID is already present in `accounts.externalUserId` belonging to some user `X`, then `X` is the correct internal user. Return `X.id` and bump their `updatedAt` and `lastLoginAt` only; do **not** overwrite their primary `users` row, because their primary may be Vercel. This is the "I signed up with Vercel, connected GitHub, and now I'm signing in directly with GitHub" path, and it is what prevents duplicate internal users from being created.
3. **No match anywhere.** Generate a new `nanoid()` and insert a fresh `users` row carrying the supplied encrypted tokens and profile fields.

`upsertUser` is the only place where a new internal user can be created during sign-in, and it is deliberately idempotent under either provider's "return visit" pattern. Returning `users.id` (not the external ID) from every branch is what lets downstream code (route handlers, task workers, the cookie layer) treat all users uniformly.

## The JWE-encrypted session cookie

The application has exactly one session cookie: `_user_session_`, defined as `SESSION_COOKIE_NAME` in `lib/session/constants.ts`. The cookie's value is a JWE produced by `lib/jwe/encrypt.ts` using `jose`'s `EncryptJWT` with the algorithm pair `alg: 'dir', enc: 'A256GCM'` and an expiration of `'1y'`. The symmetric key comes from `process.env.JWE_SECRET`, base64url-decoded. `lib/jwe/decrypt.ts` reverses the operation with `jwtDecrypt`, strips the `iat` and `exp` claims from the returned payload, and returns `undefined` (rather than throwing) on any failure so a corrupt or expired cookie is treated as "not signed in" rather than as a 500.

The decrypted payload is a `Session`:

```ts
interface Session {
  created: number                              // Date.now() at sign-in
  authProvider: 'github' | 'vercel'            // Which provider issued the cookie
  user: {
    id: string                                 // Internal users.id (nanoid)
    username: string
    email: string | undefined
    avatar: string
    name?: string
  }
}
```

The cookie is set by `saveSession` (both `lib/session/create.ts` for Vercel and `lib/session/create-github.ts` for GitHub) with the same attributes: `Path=/`, `HttpOnly`, `SameSite=Lax`, `Max-Age=1y`, and `Secure` only in production. Passing `undefined` causes `saveSession` to emit an immediate-expiry `Set-Cookie`, which is how `/api/auth/signout` clears the session.

The cookie is **not** persisted server-side; there is no session table. The cookie is the session. That choice has two consequences worth knowing:

- Revoking a session requires deleting or rotating `JWE_SECRET`; there is no per-user "log out everywhere" button.
- Any process that holds `JWE_SECRET` can both forge cookies and read existing ones, so the secret must be treated as a credential of the same class as `ENCRYPTION_KEY`.

## Reading the session: two surfaces, one decryptor

There are exactly two read paths into the cookie, both of which end at `getSessionFromCookie`:

- `lib/session/get-server-session.ts` wraps `cookies()` (Next.js's async cookie store) and is the read path used by React Server Components, server actions, and any non-route code path. It is wrapped in React's `cache(...)` so multiple `getServerSession()` calls inside a single render share one decryption.
- `lib/session/server.ts`'s `getSessionFromReq` reads from a `NextRequest`'s `cookies` API and is used inside `app/api/**` route handlers that already have the request object.

Both delegate to `getSessionFromCookie` in `lib/session/server.ts`, which is the actual JWE decryptor.

The session's `authProvider` field is the only piece of data the application uses to route the rest of auth logic. See, for example, `app/api/auth/github/disconnect/route.ts`, which refuses to disconnect GitHub when `session.authProvider === 'github'` (that would leave the user with no identity at all).

## The dual-cookie sign-in flow

The two OAuth providers share the single `_user_session_` cookie after the callback succeeds, but each provider needs to carry its own state across the redirect to the provider: state, PKCE verifier, post-auth target URL, and (for GitHub's connect variant) the internal user being linked. That state lives in short-lived auxiliary cookies that are deleted by the callback handler once consumed.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    subgraph Browser["Browser cookie jar"]
        S1["_user_session_<br/>JWE session cookie<br/>HttpOnly, 1y TTL"]
        S2["vercel_oauth_state<br/>vercel_oauth_code_verifier<br/>vercel_oauth_redirect_to<br/>HttpOnly, 10-min TTL"]
        S3a["github_auth_state<br/>github_auth_redirect_to<br/>github_auth_mode = signin<br/>HttpOnly, 10-min TTL"]
        S3b["github_auth_state<br/>github_auth_redirect_to<br/>github_auth_mode = connect<br/>github_oauth_user_id<br/>HttpOnly, 10-min TTL"]
    end

    Start(["POST /api/auth/signin/vercel<br/>or GET /api/auth/signin/github"]) --> S2
    S2 -. "provider redirect with state" .-> Provider["Vercel / GitHub authorize"]
    Provider -. "callback with code" .-> CB["/api/auth/callback/vercel<br/>or /api/auth/github/callback"]
    CB -- "decrypts state, exchanges code,<br/>calls upsertUser" --> S1
    CB -- "saveSession(...)" .-> S1
    CB -- "deletes per-flow cookies" .-> X["S2 / S3a / S3b cleared"]
    S1 --> App["Subsequent requests<br/>getServerSession / getSessionFromReq"]
```

*Diagram: the dual-cookie model. The long-lived `_user_session_` JWE cookie is the only thing that persists across requests; the per-flow OAuth state cookies are short-lived and are deleted by the callback handler.*

The aux cookies differ by provider because Vercel's PKCE flow requires one extra cookie (`vercel_oauth_code_verifier`) and because GitHub's flow has two modes, `signin` (no prior session) and `connect` (session already present, attach GitHub as a link), that the callback distinguishes by reading `github_auth_mode`. In the `connect` mode, the GitHub start route additionally writes `github_oauth_user_id = session.user.id`, which the callback reads to know which internal user to attach the new `accounts` row to.

The callback handler for GitHub still tolerates the legacy `github_oauth_*` cookie names without an `auth_mode` cookie (see `app/api/auth/github/callback/route.ts`) so old browsers mid-OAuth do not break during a deploy. New flows always write the `github_auth_*` set.

## Storing OAuth tokens

OAuth access tokens for both providers are stored in Postgres as ciphertext produced by `lib/crypto.ts`. `encrypt(plaintext)` returns an IV-and-ciphertext pair using `aes-256-cbc` with a 16-byte random IV and the 32-byte key read from `process.env.ENCRYPTION_KEY`. `decrypt(ciphertext)` reverses it and is the only place where stored tokens become usable.

`encrypt` is called from exactly two sites: inside `upsertUser` (where the primary token lands on `users.accessToken`/`refreshToken` for both providers) and inside the GitHub connect flow (where the linked-account token lands on `accounts.accessToken`). Vercel connect does not exist; Vercel cannot be linked, so Vercel tokens only ever live on the `users` row.

There is one subtlety in `upsertUser`'s call site. The GitHub primary path passes `encrypt(accessToken)` for the access token and `undefined` for `refreshToken` because GitHub's OAuth flow does not issue refresh tokens. The Vercel primary path passes both `encrypt(accessToken)` and `encrypt(refreshToken)` when present. A future GitHub OAuth app configured with refresh-token support would only need to thread `refreshToken` through `createGitHubSession`; the storage layer already accepts it.

## Reading OAuth tokens: `getOAuthToken`

`lib/session/get-oauth-token.ts` is the only function that decrypts a stored OAuth token for outbound API calls. Its provider-aware lookup is what makes the GitHub-can-be-either rule work without changing the callers:

- For `provider: 'github'` it first looks for a row in `accounts` (the linked path) and falls back to `users` (the primary path). Decryption happens on whichever row wins.
- For `provider: 'vercel'` it looks only at `users` (the primary path) because no link can exist.

The return shape is uniform:

```ts
{ accessToken: string; refreshToken: string | null; expiresAt: Date | null }
```

`expiresAt` is only populated for `accounts` rows (the schema's `accounts.expiresAt` column). For `users` rows, the function explicitly sets `expiresAt: null`; the schema does not carry one for primary accounts.

Callers of `getOAuthToken` are the three places that need to act on the user's behalf toward an upstream API: `/api/auth/info` (refresh Vercel profile data on every page load), `/api/auth/signout` (revoke the upstream token before clearing the cookie), and `/api/vercel/teams` (fetch team membership). A parallel helper, `lib/github/user-token.ts`'s `getUserGitHubToken`, exists specifically for the GitHub case and is used by the agent dispatcher to inject the user's GitHub token into the sandbox; it inlines the same "accounts-first, users-fallback" lookup but returns only the plaintext string.

## Account merge on GitHub reconnect

The GitHub connect callback in `app/api/auth/github/callback/route.ts` handles three sub-cases for the `connect` mode after exchanging the code:

1. **Same user, re-connecting.** The `(accounts.provider, accounts.externalUserId)` matches the connecting user's existing GitHub link. The encrypted token is updated in place.
2. **Conflict: GitHub account belongs to a different internal user.** This is the "another account in our system already owns this GitHub identity" case. The callback transfers every row owned by the *other* user to the *connecting* user across `tasks`, `connectors`, `accounts`, and `keys`, deletes the now-empty `users` row for the other user, and updates the existing `accounts` row to point at the connecting user. The token is overwritten at the same time.
3. **Fresh link.** Insert a new `accounts` row carrying the encrypted token and the connecting user's internal ID.

Sub-case 2 is destructive and irreversible: the previous internal user is deleted after its child rows are reparented. This is intentional (there can only be one internal user per real-world GitHub identity) but is the kind of merge a deployer should be aware of before turning on GitHub auth alongside a deployed Vercel-auth population.

## Sign-out: revoke upstream, clear cookie

`/api/auth/signout` reads the current session, fetches the upstream token via `getOAuthToken`, and revokes it at the provider before clearing the cookie. For GitHub, the revocation call hits `DELETE /applications/:client_id/token` with HTTP Basic auth using the OAuth client credentials. For Vercel, it hits `POST /api/login/oauth/token/revoke` with a form-urlencoded body containing the access token, plus the same Basic auth header. The two responses are deliberately not parsed: a revoke failure only logs to the server console and the local cookie is cleared regardless. After revocation, the route returns `{ url }` for the client to navigate to and uses `saveSession(response, undefined)` to write the immediate-expiry `Set-Cookie`.

## Environment and configuration surface

The auth subsystem reads from exactly these environment variables:

| Variable | Used by | Purpose |
| --- | --- | --- |
| `NEXT_PUBLIC_AUTH_PROVIDERS` | `lib/auth/providers.ts` | Which providers' buttons appear in the sign-in dialog (client-exposed) |
| `NEXT_PUBLIC_GITHUB_CLIENT_ID` | GitHub sign-in start, callback, and revoke routes | OAuth client ID (client-exposed) |
| `GITHUB_CLIENT_SECRET` | GitHub callback and revoke | OAuth client secret |
| `NEXT_PUBLIC_VERCEL_CLIENT_ID` | Vercel sign-in start, callback, and revoke | OAuth client ID (client-exposed) |
| `VERCEL_CLIENT_SECRET` | Vercel callback and revoke | OAuth client secret |
| `JWE_SECRET` | `lib/jwe/encrypt.ts`, `lib/jwe/decrypt.ts` | Symmetric key for the `_user_session_` cookie |
| `ENCRYPTION_KEY` | `lib/crypto.ts` | 32-byte hex key for at-rest OAuth token encryption |

Rotating `JWE_SECRET` invalidates every existing session cookie. Rotating `ENCRYPTION_KEY` requires re-running the OAuth flow for every user; the stored ciphertext cannot be decrypted under the old key. Both are fail-fast: `getEncryptionKey()` throws on a missing or wrong-length `ENCRYPTION_KEY`, and both JWE helpers throw on a missing `JWE_SECRET`.

## Failure and edge-case behavior

- **Missing provider config.** `GET /api/auth/signin/github` redirects to `/?error=github_not_configured` if `NEXT_PUBLIC_GITHUB_CLIENT_ID` is unset. The Vercel sign-in route constructs an `OAuth2Client` with an empty string and will fail downstream; there is no explicit guard.
- **State mismatch.** Both callbacks refuse to complete the exchange when the incoming `state` does not match the stored state, when the code is missing, or when the post-auth `redirect_to` cookie is missing. They return HTTP 400 in those cases.
- **Bad token exchange.** The GitHub callback returns 400 if GitHub's token endpoint responds non-OK or omits `access_token`. The Vercel callback catches `validateAuthorizationCode`'s rejection and returns 400.
- **JWE decrypt failure.** `decryptJWE` swallows the error and returns `undefined`. Callers treat that as "not signed in", so a tampered or expired cookie degrades to an anonymous request rather than a 500.
- **Corrupt encrypted token.** `decrypt` throws. Callers that hit this path log the error and return `null` so the route handler returns 401 (see `/api/vercel/teams` for the canonical example).
- **Disconnecting the primary provider.** `/api/auth/github/disconnect` returns 400 when `session.authProvider === 'github'`; the user would be left without any identity if it succeeded.

## Extension points

- **Adding a third primary provider** requires a new value in the `users.provider` enum and the corresponding Zod schema, a new sign-in start route under `app/api/auth/signin/`, a new callback route, a new `createSession` helper in `lib/session/`, and an entry in `getEnabledAuthProviders`. The cookie shape (`authProvider`) and the JWE helpers are provider-agnostic, so only the OAuth handshake needs duplicating.
- **Adding a new linked provider** requires extending the `accounts.provider` enum (currently locked to `'github'`), threading the new provider through `upsertUser`'s GitHub-only branch and through `getOAuthToken`'s lookup, and adding a "connect" UI flow analogous to `/api/auth/signin/github` for the new provider.
- **Adding token refresh** would require threading refresh-token storage through `upsertUser` (already supported for Vercel; not yet wired for GitHub), teaching `getOAuthToken` to surface `expiresAt`, and teaching the few callers that currently assume a static token to refresh on 401.
