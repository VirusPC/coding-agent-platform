# Files

- [GitHub OAuth](github-oauth.md) - The GitHub side of the auth subsystem — OAuth app registration, scopes, the sign-in and connect routes, the callback's two-mode dispatch, AES-256-CBC encryption of stored tokens, and how the access token is decrypted for outbound API calls.
- [Vercel AI Gateway](vercel-ai-gateway.md) - The two faces of the AI Gateway integration — AI SDK 5 generators for branch names, task titles, and commit messages, and the sandbox-side routing of the Claude and Codex CLIs through https://ai-gateway.vercel.sh via shared AI_GATEWAY_API_KEY credentials with structured fallback paths when the key is absent.
- [Vercel OAuth](vercel-oauth.md) - The Vercel side of the auth subsystem — OAuth app registration, the PKCE (S256) handshake via arctic, default scopes, callback exchange, the AES-256-CBC at-rest encryption of the primary Vercel token, and how /api/auth/info rehydrates the session on every page load.
- [Integration: Vercel Sandbox](vercel-sandbox.md)
