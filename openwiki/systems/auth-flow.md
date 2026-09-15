---
type: system
title: Auth API Surface
description: The HTTP routes under app/api/auth/ that drive sign-in (GitHub, Vercel), callback, session info, rate-limit, GitHub status, GitHub disconnect, and sign-out, plus the client-side hooks that call them and the three GitHub cookie sets they read.
tags: [auth, api, oauth, github, vercel, signin, callback, signout, session, cookies, arctic, pkce, rate-limit]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-6e7fdb664c7401491a0a2550
    resource: repo://app/api/auth/callback/vercel/route.ts
  - id: openwiki-source-1745d192079a6690c9cd73cc
    resource: repo://app/api/auth/github/callback/route.ts
  - id: openwiki-source-62cac3fdfdcec4709a5b5641
    resource: repo://app/api/auth/github/disconnect/route.ts
  - id: openwiki-source-42ea7e86aea38ae28a926b77
    resource: repo://app/api/auth/github/signin/route.ts
  - id: openwiki-source-c348957a0838bc5e1406466a
    resource: repo://app/api/auth/github/status/route.ts
  - id: openwiki-source-09ad3bfa51b2ad2786247a20
    resource: repo://app/api/auth/info/route.ts
  - id: openwiki-source-c8c5a4569049020ef0e7849f
    resource: repo://app/api/auth/rate-limit/route.ts
  - id: openwiki-source-9ae4d3c854e5a4c6ef24eeb3
    resource: repo://app/api/auth/signin/github/route.ts
  - id: openwiki-source-552741c58d746b8418a79f4f
    resource: repo://app/api/auth/signin/vercel/route.ts
  - id: openwiki-source-0dd0bd54c06ca50efe9ecefa
    resource: repo://app/api/auth/signout/route.ts
  - id: openwiki-source-a0047794cd389c338a104939
    resource: repo://components/auth/session-provider.tsx
  - id: openwiki-source-da3f9ac8030f7d69eb8dbb85
    resource: repo://components/auth/sign-in.tsx
  - id: openwiki-source-1c2e4ec9ff65ebebe383abb1
    resource: repo://components/auth/sign-out.tsx
  - id: openwiki-source-5387b2d865c5b9ef618373cf
    resource: repo://components/auth/user.tsx
  - id: openwiki-source-7b713bd3d987acffddc4c106
    resource: repo://components/home-page-content.tsx
  - id: openwiki-source-987b65089d49a0b1a1abb119
    resource: repo://components/repo-selector.tsx
  - id: openwiki-source-f4a535c4711843f9246eaee1
    resource: repo://lib/auth/providers.ts
  - id: openwiki-source-7ece006b3c5e6a3e70e9b390
    resource: repo://lib/utils/rate-limit.ts
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Auth API Surface

This page is the route map for everything that lives under `app/api/auth/`: the two sign-in entrypoints, the two callback handlers, the session-info probe, the rate-limit endpoint, the GitHub link-management endpoints, and the sign-out route. It exists so a coding agent adding a provider or changing the flow can find the right file without spelunking through the consumer chain. The concept material that ties these routes together (the JWE cookie, the `users`/`accounts` split, the `arctic` PKCE dance) lives in [Authentication & Sessions](../concepts/auth-and-sessions.md); the per-provider integration details are in [GitHub OAuth](../integrations/github-oauth.md) and [Vercel OAuth](../integrations/vercel-oauth.md).

## Route inventory

Every route is a Next.js App Router handler under `app/api/auth/`. Methods are the only HTTP verbs exported.

| Route | Methods | Purpose |
| --- | --- | --- |
| `app/api/auth/signin/vercel/route.ts` | `POST` | Start the Vercel PKCE flow; returns the authorize URL as JSON. |
| `app/api/auth/signin/github/route.ts` | `GET` | Unified signin/connect entrypoint; redirects to GitHub with `signin` or `connect` mode selected by current session. |
| `app/api/auth/github/signin/route.ts` | `GET`, `POST` | Legacy connect-only entrypoint; requires an existing session. GET redirects, POST returns `{ url }`. |
| `app/api/auth/callback/vercel/route.ts` | `GET` | Vercel redirect target; exchanges code via `arctic`, creates a session, writes the JWE cookie. |
| `app/api/auth/github/callback/route.ts` | `GET` | GitHub redirect target; dispatches on `github_auth_mode` to signin or connect. |
| `app/api/auth/info/route.ts` | `GET` | Returns the current session's `{ user, authProvider }`; rehydrates Vercel users from the stored token. |
| `app/api/auth/rate-limit/route.ts` | `GET` | Returns `{ allowed, remaining, used, total, resetAt }` for the current user. |
| `app/api/auth/github/status/route.ts` | `GET` | Returns `{ connected, username?, connectedAt? }` for the current session's GitHub link. |
| `app/api/auth/github/disconnect/route.ts` | `POST` | Deletes the linked `accounts` row for GitHub; refuses when GitHub is the primary auth. |
| `app/api/auth/signout/route.ts` | `GET` | Revokes the upstream token, clears the session cookie, returns `{ url }` for post-signout navigation. |

The single client component that drives most of this surface is `components/auth/session-provider.tsx`, which polls `/api/auth/info` and `/api/auth/github/status` every 60 seconds and on `window.focus`. The user-dropdown UI in `components/auth/sign-out.tsx` adds `/api/auth/rate-limit` and `/api/auth/github/disconnect` to that set, and `/api/auth/github/signin` is also navigated to directly from the connect button.

## The Vercel flow (`signin/vercel` and `callback/vercel`)

The Vercel start route is a `POST` because the `arctic` `OAuth2Client.createAuthorizationURLWithPKCE` returns a URL that the client must navigate to; returning JSON and letting the browser set `window.location` is the only ergonomic way to keep the call site inside a React click handler ([`lib/session/redirect-to-sign-in.ts`](repo://lib/session/redirect-to-sign-in.ts)).

```ts
// app/api/auth/signin/vercel/route.ts
const client = new OAuth2Client(
  process.env.NEXT_PUBLIC_VERCEL_CLIENT_ID ?? '',
  process.env.VERCEL_CLIENT_SECRET ?? '',
  `${req.nextUrl.origin}/api/auth/callback/vercel`,
)

const state = generateState()
const verifier = generateCodeVerifier()
const url = client.createAuthorizationURLWithPKCE(
  'https://vercel.com/oauth/authorize',
  state,
  CodeChallengeMethod.S256,
  verifier,
  [], // Vercel uses default scopes
)
```

Three short-lived cookies carry the handshake state across the redirect:

| Cookie | Purpose |
| --- | --- |
| `vercel_oauth_state` | CSRF token to compare in the callback. |
| `vercel_oauth_code_verifier` | PKCE S256 verifier for the token exchange. |
| `vercel_oauth_redirect_to` | Post-auth `next` target (relative URL only, validated by [`isRelativeUrl`](repo://lib/utils/is-relative-url.ts)). |

All three are set with `HttpOnly`, `SameSite=Lax`, `Secure` in production, and a 10-minute `maxAge`. The `arctic` library supplies `OAuth2Client`, `generateState`, `generateCodeVerifier`, and `CodeChallengeMethod.S256`; the route does not implement PKCE by hand. The empty `scopes` argument is intentional and commented inline — Vercel grants the integration's full configured permission set by default.

The callback at [`app/api/auth/callback/vercel/route.ts`](repo://app/api/auth/callback/vercel/route.ts) reads `code` + `state` from the query string and the three cookies from the jar. Missing or mismatched state returns 400 without deleting the cookies, so a retry can succeed once the start route reissues them. The token exchange is the only `arctic` call that has to pass Vercel's bespoke token endpoint explicitly, because `arctic` does not ship a Vercel adapter:

```ts
tokens = await client.validateAuthorizationCode(
  'https://vercel.com/api/login/oauth/token',
  code,
  storedVerifier,
)
```

`createSession({ accessToken, expiresAt, refreshToken })` from [`lib/session/create.ts`](repo://lib/session/create.ts) is what hits Vercel for the profile (with a `fetchUser` fallback to `vercel.com/api/www/user` if `api.vercel.com/v2/user` is unreachable) and calls `upsertUser`. The 302 `Response` with the post-auth `Location` is built *before* `createSession` runs so the same response can carry both the `Set-Cookie: _user_session_=…` header and the navigation. If `createSession` returns `undefined` the route returns 500 `Failed to create session` and discards the 302; the cookies are not deleted, so the user can reattempt by visiting the start route again.

## The GitHub flow: three entrypoints, one callback

GitHub has three start routes that exist in parallel and a single callback that tolerates any of them. The callback at [`app/api/auth/github/callback/route.ts`](repo://app/api/auth/github/callback/route.ts) is the most complex route in the subsystem because it has to dispatch on a cookie-supplied mode and then handle three connect sub-cases.

### The unified entrypoint: `GET /api/auth/signin/github`

[`app/api/auth/signin/github/route.ts`](repo://app/api/auth/signin/github/route.ts) is the modern path. It picks the mode by checking for an existing session with `getSessionFromReq`:

```ts
const isSignInFlow = !session?.user
const authMode = isSignInFlow ? 'signin' : 'connect'
```

In `signin` mode the route writes three cookies and redirects directly to GitHub. In `connect` mode it additionally writes `github_oauth_user_id = session.user.id`, appends `?github_connected=true` to the post-auth URL (read by `components/home-page-content.tsx` to show a toast), and writes the same `github_auth_*` cookies. The three (or four) cookies are:

| Cookie | Mode | Purpose |
| --- | --- | --- |
| `github_auth_state` | both | CSRF token |
| `github_auth_redirect_to` | both | Post-auth `next` target |
| `github_auth_mode` | both | `'signin'` or `'connect'`; the callback dispatches on this |
| `github_oauth_user_id` | connect only | Internal `users.id` to attach the new `accounts` row to |

`github_auth_mode` is the disambiguator that lets the callback know which cookie set to read and which sub-flow to run.

### The legacy entrypoints: `GET` and `POST /api/auth/github/signin`

[`app/api/auth/github/signin/route.ts`](repo://app/api/auth/github/signin/route.ts) predates the unified route and is still live for back-compat. It only writes the `github_oauth_*` cookie set (no `auth_mode`) and treats every caller as a connect flow:

```ts
for (const [key, value] of [
  [`github_oauth_redirect_to`, redirectTo],
  [`github_oauth_state`, state],
  [`github_oauth_user_id`, session.user.id], // Store Vercel user ID
]) store.set(key, value, ...)
```

The route exports both `GET` and `POST`. `GET` redirects to GitHub directly (used by `components/auth/sign-out.tsx` and `components/home-page-content.tsx` for the "Connect" button). `POST` returns `{ url }` as JSON for callers that need to navigate programmatically. Both require a current session — an unauthenticated `GET` redirects to `/` and an unauthenticated `POST` returns 401.

### The callback: `GET /api/auth/github/callback`

[`app/api/auth/github/callback/route.ts`](repo://app/api/auth/github/callback/route.ts) is the dispatcher. It picks the cookie set by checking whether `github_auth_mode` is present:

```ts
const authMode = cookieStore.get(`github_auth_mode`)?.value ?? null
const storedState = cookieStore.get(authMode ? `github_auth_state` : `github_oauth_state`)?.value ?? null
const storedRedirectTo =
  cookieStore.get(authMode ? `github_auth_redirect_to` : `github_oauth_redirect_to`)?.value ?? null
```

State mismatch returns 400 `Invalid OAuth state`. For `signin` mode the route only needs `state` + `redirect_to`; for `connect` (and the legacy no-mode path) it additionally requires `github_oauth_user_id`.

After a successful `POST https://github.com/login/oauth/access_token` (JSON body, JSON response, with `Accept: application/json`) and a `GET https://api.github.com/user` profile fetch, the route branches:

- **`signin` mode.** Calls `createGitHubSession(accessToken, scope)` from [`lib/session/create-github.ts`](repo://lib/session/create-github.ts), which falls back to `/user/emails` for a primary verified address when the public profile has no `email`, then `upsertUser({ provider: 'github', externalId, accessToken: encrypt(accessToken), refreshToken: undefined, scope, … })`. The encrypted token lands on `users.accessToken`. `saveSession(response, session)` writes the JWE cookie and the 302 to `storedRedirectTo`, then the three `github_auth_*` cookies are deleted.
- **`connect` mode.** Encrypts the new token with `encrypt(token)` from [`lib/crypto.ts`](repo://lib/crypto.ts) and writes it to `accounts`. The route then switches on three sub-cases after a `SELECT FROM accounts WHERE provider='github' AND externalUserId={id}`:
  1. **Same user.** The existing row's `userId` matches the connecting `users.id`; update `accessToken`, `scope`, `username`, `updatedAt` in place.
  2. **Conflict (GitHub identity belongs to another internal user).** Reparent every row owned by the *other* user onto the connecting user (`tasks`, `connectors`, `accounts`, `keys`), delete the now-empty old `users` row, and update the existing `accounts` row's `userId` and token. The previous user's own `users.accessToken` is never touched.
  3. **Fresh link.** `INSERT INTO accounts` with a `nanoid()` id, the encrypted token, the GitHub numeric ID, the granted scope, and the GitHub `login`.

The destructive merge in sub-case 2 is intentional but irreversible — it is what prevents one real-world GitHub identity from spawning multiple internal users.

The legacy no-`auth_mode` branch deletes the `github_oauth_*` cookies instead of the `github_auth_*` set, so an old client mid-flow does not break during a deploy.

## Session, status, and rate-limit

[`app/api/auth/info/route.ts`](repo://app/api/auth/info/route.ts) is the route `components/auth/session-provider.tsx` polls. Its contract is provider-aware:

- **No session.** Returns `{ user: undefined }` and emits the immediate-expiry `Set-Cookie` for `_user_session_` via `saveSession(response, undefined)` (which clears any stale value).
- **GitHub primary.** Returns the existing session verbatim; no rehydration.
- **Vercel primary.** Reads the encrypted token via `getOAuthToken(userId, 'vercel')`, decrypts with `lib/crypto.ts`, and re-runs `createSession({ accessToken, expiresAt })` to refresh profile data. The refreshed session is re-encrypted and re-issued with `saveSession`.

This rehydration is why a Vercel user who changes their profile sees the new name and email without re-signing-in: the cookie carries the original snapshot, but `/api/auth/info` returns the current Vercel data on every page load.

[`app/api/auth/github/status/route.ts`](repo://app/api/auth/github/status/route.ts) returns one of three shapes for the current session:

- `{ connected: false }` — no session, or session has no GitHub link or primary.
- `{ connected: true, username, connectedAt }` — an `accounts` row with `provider = 'github'` exists for the user.
- `{ connected: true, username, connectedAt }` — the user's `users` row has `provider = 'github'`.

The two-bucket check is what lets the UI render the same "Connected as …" badge whether GitHub is the user's primary identity or a linked one.

[`app/api/auth/rate-limit/route.ts`](repo://app/api/auth/rate-limit/route.ts) calls [`checkRateLimit(session.user.id)`](repo://lib/utils/rate-limit.ts), which sums today's non-soft-deleted `tasks` and `taskMessages.role = 'user'` rows and returns `{ allowed, remaining, total, resetAt }`. `resetAt` is the next UTC midnight. The dropdown renders this as `{remaining}/{total} messages remaining today`.

## Disconnect and sign-out

[`app/api/auth/github/disconnect/route.ts`](repo://app/api/auth/github/disconnect/route.ts) is a `POST` that deletes the `accounts` row whose `userId` matches the current session and whose `provider` is `'github'`. It refuses with 400 `Cannot disconnect primary authentication method` when `session.authProvider === 'github'` — removing the primary identity would leave a session whose `users.accessToken` is the only way to act on the user's behalf, and there is no fallback path. The component layer only renders the Disconnect menu item when `authProvider === 'vercel'`, so the route guard is a second line of defense.

[`app/api/auth/signout/route.ts`](repo://app/api/auth/signout/route.ts) is a `GET` that walks through four steps:

1. Read the current session with `getSessionFromReq`.
2. Fetch the upstream token via `getOAuthToken(userId, provider)` — provider is `session.authProvider`.
3. Revoke the token at the provider:
   - **GitHub.** `DELETE https://api.github.com/applications/{client_id}/token` with HTTP Basic auth built from `NEXT_PUBLIC_GITHUB_CLIENT_ID:GITHUB_CLIENT_SECRET` and a JSON body of `{ access_token }`.
   - **Vercel.** `POST https://vercel.com/api/login/oauth/token/revoke` with a form-urlencoded `{ token }` body and the same Basic auth header.
4. Return `{ url }` (the validated `next` query parameter, or `/`) and call `saveSession(response, undefined)` to write the immediate-expiry `Set-Cookie` that clears `_user_session_`.

The revoke responses are deliberately not parsed: a revoke failure only logs to the server console and the local cookie is cleared regardless. The client-side [`redirectToSignOut`](repo://lib/session/redirect-to-sign-out.ts) reads `{ url }` and sets `window.location = url`.

## The three GitHub cookie sets

The callback handler tolerates three distinct cookie shapes. They share the `HttpOnly`, `SameSite=Lax`, `Secure`-in-production, 10-minute TTL profile; they differ in name and in what the callback reads.

| Cookie name | Written by | Read by callback as | Meaning |
| --- | --- | --- | --- |
| `github_auth_state` | `signin/github` (unified) | `storedState` when `authMode` set | Unified CSRF state |
| `github_auth_redirect_to` | `signin/github` (unified) | `storedRedirectTo` when `authMode` set | Unified post-auth target |
| `github_auth_mode` | `signin/github` (unified) | mode disambiguator | `'signin'` or `'connect'` |
| `github_oauth_state` | `github/signin` (legacy) | `storedState` when `authMode` absent | Legacy CSRF state |
| `github_oauth_redirect_to` | `github/signin` (legacy) | `storedRedirectTo` when `authMode` absent | Legacy post-auth target |
| `github_oauth_user_id` | both unified (`connect`) and legacy | connecting user's `users.id` | Required for any connect flow |

The legacy set exists because the unified route was added by *extending* the callback rather than replacing it, so a browser mid-OAuth during a deploy does not break. New client code should call the unified route; the legacy entries are kept only for the connect button in the dropdown and for any bookmarked URLs that point at the old path. The presence of `github_auth_mode` is the discriminator the callback uses to know which set to read.

## Client integration

`components/auth/session-provider.tsx` is the polling loop that keeps the Jotai `sessionAtom` and `githubConnectionAtom` warm:

```ts
// pseudo-flow inside SessionProvider
await Promise.all([
  fetch('/api/auth/info'),         // → setSession(data)
  fetch('/api/auth/github/status'),// → setGitHubConnection(data)
])
// refresh every 60s and on window focus
```

The auth UI split between three components is by sign-in state:

- **`components/auth/user.tsx`** picks `SignOut` or `SignIn` based on the session atom (with a SSR-friendly prop fallback while the atom is initializing).
- **`components/auth/sign-in.tsx`** renders the provider dialog; `getEnabledAuthProviders()` decides which buttons appear. The Vercel button calls `redirectToSignIn()` (POST to `/api/auth/signin/vercel` and navigates to the returned URL). The GitHub button sets `window.location.href = '/api/auth/signin/github'`.
- **`components/auth/sign-out.tsx`** renders the avatar dropdown. It POSTs to `/api/auth/github/disconnect` for the Disconnect item, GETs `/api/auth/signout` (via `redirectToSignOut`) for Log Out, navigates to `/api/auth/github/signin` for Connect, and GETs `/api/auth/rate-limit` on every dropdown open to refresh the remaining-messages count.

The unified sign-in route's `connect` branch appends `?github_connected=true` to the post-auth URL; `components/home-page-content.tsx` reads that parameter on mount, surfaces a success toast, then strips it from the URL so a page refresh does not re-trigger the toast.

`components/repo-selector.tsx` is a secondary consumer that POSTs to `/api/auth/github/disconnect` whenever a `/api/github/user` or `/api/github/orgs` call returns 401/403 — that path is the self-healing branch that drops a stale GitHub link without waiting for user interaction.

## Failure modes and invariants

The auth API surface has a handful of invariants that show up across multiple routes; documenting them here so a future change does not break the contract.

- **State mismatch returns 400 without deleting cookies.** Both callbacks preserve the per-flow cookies on a 400 so a retry from the same start route has a chance to succeed.
- **Callback 500s do not delete cookies either.** Vercel's `createSession` failure path and GitHub's connect sub-case 2 errors leave the per-flow cookies in place. The user lands on a 500 page; the next visit to the start route begins a fresh flow.
- **`getEnabledAuthProviders` is the only deploy-time filter.** It reads `NEXT_PUBLIC_AUTH_PROVIDERS` (a comma-separated list, defaulting to `github`) and is consumed by both sign-in dialogs and the connect menu. A deploy with only one provider enabled hides the other button, but the routes still exist and still respond — there is no per-route allow-list.
- **`GET /api/auth/signin/github` redirects to `/?error=github_not_configured` when `NEXT_PUBLIC_GITHUB_CLIENT_ID` is unset.** The POST variant and the callback return 500 instead.
- **`GET /api/auth/signout` always clears the cookie.** The revoke step is best-effort; failure is logged and the user is signed out locally regardless.
- **`POST /api/auth/github/disconnect` is the only place that mutates `accounts`.** No other route deletes or rewrites the row. The unique index `(userId, provider)` on `accounts` is what enforces "one GitHub link per internal user".
- **No session-table backing.** The `_user_session_` cookie is the session; rotating `JWE_SECRET` is the only way to invalidate every cookie at once.
- **All session writes go through one of the two `saveSession` helpers.** [`lib/session/create.ts`](repo://lib/session/create.ts) (Vercel) and [`lib/session/create-github.ts`](repo://lib/session/create-github.ts) (GitHub) emit the identical `Set-Cookie` shape (`Path=/`, `HttpOnly`, `SameSite=Lax`, `Max-Age=1y`, `Secure` in production). `/api/auth/info` picks the helper based on `session.authProvider`.

## Extension points

- **Adding a third primary OAuth provider** requires one new sign-in start route under `app/api/auth/signin/`, one new callback route, one new `createSession` helper in `lib/session/`, one entry in `getEnabledAuthProviders`, and a new value in the `users.provider` enum (and Zod schema) in [`lib/db/schema.ts`](repo://lib/db/schema.ts). The cookie shape, the JWE helpers, and `/api/auth/info` are provider-agnostic. The sign-out revoke step is the only call site that needs a third `else if` for the new provider.
- **Adding a second linked provider** (mirroring GitHub) requires a new value in the `accounts.provider` enum, a parallel `signin/<provider>` route that picks `signin` vs. `connect` from the session, and a generalization of the connect branch in `app/api/auth/github/callback/route.ts`. The current `accounts` provider enum is hard-coded to `'github'` in [`lib/db/schema.ts`](repo://lib/db/schema.ts#L265-L269), so widening it is the first schema change. The connect cookie contract (`auth_mode`, `auth_state`, `auth_redirect_to`, `oauth_user_id`) is provider-agnostic.
- **Deprecating the legacy GitHub entrypoint.** Once no consumer references `/api/auth/github/signin`, delete that route and the legacy `github_oauth_*` branch in the callback. The unified mode is the long-term path.
- **Adding a token-refresh path** would mean threading `expiresAt` and `refreshToken` through `getOAuthToken`'s return shape (already done for `accounts`, only `null` for `users`), teaching `/api/auth/info` (and the agents dispatcher that uses the same helper) to refresh on a 401, and persisting refresh tokens for GitHub by extending `createGitHubSession`. The schema already accepts both columns.
