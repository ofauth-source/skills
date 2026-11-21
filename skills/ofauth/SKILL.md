---
name: ofauth
description: Integrates OFAuth V3 into an application so its users can connect OnlyFans accounts and the app can call OnlyFans data through OFAuth. Use when the user asks to add OFAuth, OnlyFans login or "Connect OnlyFans", OnlyFans API access, OnlyFans webhooks or Realtime events, Vault+ media storage, or a private app for their own OFAuth workspace accounts, or when the code uses link.ofauth.com/v3/authorize, api.ofauth.com/v3, /v3/link/token, /v3/link/register, /v3/access/*, /v3/linked-accounts, /v3/vault-plus/*, /v3/realtime, @ofauth/onlyfans-sdk (OFAuthLink, OFAuthClient), ofext_sandbox_, ofext_test_, or ofext_live_ client IDs, or the x-linked-account-id header. Not for maintaining a V2 integration (API keys, /v2 routes, Link embed).
license: Proprietary. Free to use for building OFAuth integrations.
metadata:
  author: OFAuth
  version: "1.1.1"
  homepage: https://docs.ofauth.com/v3/agents/skills
---

# OFAuth V3 integration

You are adding OFAuth V3 to an application. The app sends an account owner through an OAuth 2.0 consent flow, receives bearer tokens on its backend, and calls typed OnlyFans operations through OFAuth for the accounts the owner shared.

OFAuth is an unofficial, independent service. It is not affiliated with, endorsed by, or operated by OnlyFans. Never write code comments, UI copy, or docs that imply an affiliation.

Core facts:

- OAuth 2.0 Authorization Code with PKCE `S256`, confidential backend clients only. Authorize in the browser at `GET https://link.ofauth.com/v3/authorize`. Issuer: `https://api.ofauth.com/v3/link`.
- Exchange, refresh, and revoke at `POST https://api.ofauth.com/v3/link/token` and `/v3/link/revoke` with a form body (`application/x-www-form-urlencoded`) and exactly one of `client_secret_basic` or `client_secret_post`.
- Access tokens are opaque and short-lived (15 minutes; always use `expires_in`). Every successful code exchange returns a refresh token; no `offline_access` scope exists. Refresh returns a new access token and the same refresh token. Refresh tokens expire after 90 days without a successful refresh.
- Find accounts with `GET https://api.ofauth.com/v3/linked-accounts`. Call `https://api.ofauth.com/v3/access/*` and `/v3/vault-plus/*` with `Authorization: Bearer <access_token>` and `x-linked-account-id: <id>`. API keys are rejected. Each operation needs a scope.
- Developer integrations have one client per environment. The client ID (`ofext_sandbox_...`, `ofext_test_...`, `ofext_live_...`) selects the environment. Nothing crosses environments.
- Consent is a top-level browser page. Never an iframe or `postMessage`.
- Prefer the official V3 SDK for the app's language (TypeScript `@ofauth/onlyfans-sdk` 4.x with `OFAuthLink` and `OFAuthClient`; Python `onlyfans-sdk` 4.x; both currently release candidates). The docs say to use plain HTTP for Go and C# until their release candidates are published. Plain HTTP is always fine. SDKs do not store or refresh tokens; the app or its auth framework owns that.
- Docs: https://docs.ofauth.com/v3 (index for agents: https://docs.ofauth.com/llms.txt; append `.md` to any page for markdown). If a detail is not in this skill or the docs, ask the user or tell them to contact support@ofauth.com. Do not invent routes, fields, scopes, headers, or behavior.

## Workflow

Keep a short written plan and show it to the user before large edits. Reuse the codebase's HTTP client, config loader, secret storage, database layer, OAuth framework, and job runner.

### 1. Choose the app type and credentials

Decide which kind of app this is. A **Developer integration** is a platform that onboards other people's accounts: the user creates it in the OFAuth dashboard, gets a client ID and secret per environment (Sandbox, Test, Live), registers redirect URIs there, and pays with a Developer plan. An **Operator private app** is a self-hosted app or script acting on the accounts in the user's own Operator workspace: it registers itself at `POST https://api.ofauth.com/v3/link/register` (RFC 7591), uses Operator Access scopes (`access:read`, `access:write`, `access:destructive`, `connections:manage`), sends `Idempotency-Key` on every write, runs requests directly with no browser confirmation step, and has no webhooks, Vault+, or Realtime. If unsure, ask. See [references/oauth.md](references/oauth.md#operator-private-apps).

For a Developer integration, read credentials from environment variables such as `OFAUTH_SANDBOX_CLIENT_ID` and `OFAUTH_SANDBOX_CLIENT_SECRET` (match the codebase's naming). Never write real secrets into files, fixtures, or chat. Add placeholders to example env files and tell the user which values to fill in from the dashboard. Do not edit existing values in `.env` files. See [references/environments.md](references/environments.md).

### 2. Start in Sandbox

Build and test against the Sandbox client first. Sandbox returns sample data and never contacts OnlyFans. Loopback redirect URIs (`http://127.0.0.1:<port>/...`) work there; `http://localhost` does not (Developer clients).

### 3. Implement authorize, callback, and token exchange

On the backend: create a random `state` and PKCE verifier, store them server-side bound to the app user and browser session for one use, and redirect to the authorization URL with `response_type=code`, `client_id`, `redirect_uri`, `state`, `code_challenge`, `code_challenge_method=S256`, and `scope`. Send no other parameters (no `purpose`; Live shows the purpose from the approved app configuration). In the callback, consume the state once, handle `error`, check `iss`, then exchange the code within 5 minutes. See [references/oauth.md](references/oauth.md).

### 4. Store tokens and map linked accounts

Store the refresh token server-side, encrypted at rest, under the app user and the client ID (environment). Track access-token expiry from `expires_in`. Discover accounts with `GET /v3/linked-accounts`, following `next_cursor` while `has_more` is `true`; do not treat the callback or token response as the account list. Let the user choose the account. Protect against stale writes when a user reconnects or disconnects. See [references/oauth.md](references/oauth.md#storage-model) and [references/access.md](references/access.md#linked-accounts).

### 5. Call Access with the right scopes

Request only the scopes the app's operations need. Send the bearer token and `x-linked-account-id` on every Access and Vault+ call; refresh shortly before expiry. Confirm each operation's path and scope on its API reference page. See [references/access.md](references/access.md). For stored media, see [references/vault-plus.md](references/vault-plus.md).

### 6. Add a webhook receiver (Developer integrations)

The user adds a destination per environment in the dashboard and gives you its signing secret through an env var. Verify Standard Webhooks signatures on the raw body before parsing, answer the signed verification challenge, save the event and deduplicate on `eventId`, return `2xx` promptly, and handle `authorization.*` and `linked_account.*` lifecycle events. See [references/webhooks.md](references/webhooks.md).

### 7. Handle errors by recovery action

Branch on `recovery.action` from the API error body, not on HTTP status alone. Token endpoint errors follow RFC 6749. Log `requestId`. Never automatically repeat a write whose outcome is unknown. See [references/errors.md](references/errors.md).

### 8. Move to Test, then Live

Test uses real OnlyFans accounts the integration owner designates, with a daily quota. Live needs an active Developer plan, Live approval from the dashboard, and HTTPS redirect URIs with a domain hostname. See [references/environments.md](references/environments.md).

## Security rules

- Keep the client secret, PKCE verifier, state records, and access and refresh tokens on the server. Send only the authorization URL to browsers and native apps. Native apps open it in `ASWebAuthenticationSession` (iOS) or a Custom Tab (Android) and never exchange tokens themselves.
- Bind `state` to the initiating app user and browser session and consume it exactly once. Reject missing, changed, expired, or reused state. Validate `iss` on the callback.
- Refresh needs no lock: concurrent refreshes and retries after a lost response are safe. On `invalid_grant`, mark the user for re-authorization.
- Never send an access token to a browser. For Realtime in a web page, create a stream URL on the server.
- Do not log tokens, secrets, stream URLs, or full webhook bodies. Keep tokens out of URLs. Send client secrets and refresh tokens only to the OFAuth OAuth endpoints.
- Store the environment (client ID) with every token and linked account id. A Sandbox token cannot reach a Live account.
- Verify every webhook signature before trusting the body. Keep sample `test: true` deliveries out of customer records, and use `environment` to separate Sandbox, Test, and Live state.

## References

- [references/oauth.md](references/oauth.md): authorization request, redirect URIs, callback, token exchange, refresh, revoke, ending an authorization, discovery, Operator private apps.
- [references/access.md](references/access.md): headers, SDK, scopes, linked accounts, reconnect, pagination, rate limits, media.
- [references/webhooks.md](references/webhooks.md): destinations, verification, envelope, lifecycle and Realtime events, Server-Sent Events stream.
- [references/vault-plus.md](references/vault-plus.md): Vault+ routes, scopes, settings, add and sync semantics.
- [references/environments.md](references/environments.md): Sandbox, Test, Live, configuration, go-live checklist.
- [references/errors.md](references/errors.md): API and OAuth error codes, recovery actions, handling pattern.
