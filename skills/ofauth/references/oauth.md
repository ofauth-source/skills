# OAuth: authorize, tokens, refresh, revoke

Full reference: https://docs.ofauth.com/v3/api-reference/link/authorize and https://docs.ofauth.com/v3/api-reference/link/tokens. Security model: https://docs.ofauth.com/v3/reference/security.

## Contents

- [Endpoints](#endpoints)
- [Authorization request](#authorization-request)
- [Redirect URI rules](#redirect-uri-rules)
- [Browser surface](#browser-surface)
- [Callback](#callback)
- [Token endpoint](#token-endpoint)
- [Token response](#token-response)
- [Refresh](#refresh)
- [Storage model](#storage-model)
- [Requesting more scopes or accounts](#requesting-more-scopes-or-accounts)
- [Revoke and disconnect](#revoke-and-disconnect)
- [User info](#user-info)
- [Discovery](#discovery)
- [Operator private apps](#operator-private-apps)

## Endpoints

| Purpose | Endpoint |
| --- | --- |
| Authorize (browser) | `GET https://link.ofauth.com/v3/authorize` |
| Token | `POST https://api.ofauth.com/v3/link/token` |
| Revoke (RFC 7009) | `POST https://api.ofauth.com/v3/link/revoke` |
| Current authorization | `GET https://api.ofauth.com/v3/authorization` |
| End whole authorization | `DELETE https://api.ofauth.com/v3/authorization` |
| User info | `GET https://api.ofauth.com/v3/link/userinfo` |
| Discovery (RFC 8414) | `GET https://api.ofauth.com/.well-known/oauth-authorization-server/v3/link` |
| Operator private app registration (RFC 7591) | `POST https://api.ofauth.com/v3/link/register` |

The issuer is `https://api.ofauth.com/v3/link`.

## Authorization request

Build the URL on the backend.

| Parameter | Requirement |
| --- | --- |
| `response_type` | `code` |
| `client_id` | The client ID for the target environment. It selects the environment. |
| `redirect_uri` | An exact registered redirect URI for that client. |
| `state` | Random, one use. Returned on the redirect. |
| `code_challenge` | base64url SHA-256 of the PKCE verifier (43 characters). |
| `code_challenge_method` | `S256` |
| `scope` | Space-separated scopes. See [access.md](access.md#scopes). |

Each parameter may appear once. Unknown or repeated parameters return `invalid_request`, so do not add `purpose`, `offline_access`, or other extras. Live consent shows the purpose from the approved app configuration. Do not send `resource`; if an OAuth library always sends it, it must be `https://api.ofauth.com/v3`.

With the TypeScript SDK:

```typescript
import { OFAuthLink } from "@ofauth/onlyfans-sdk"

const link = new OFAuthLink({ clientId, clientSecret, redirectUri })
const attempt = await link.createAuthorizationUrl({ scopes: ["profile:read"] })
// Persist attempt.state and attempt.codeVerifier with the trusted app user,
// browser session, expiry, and registration version. Then redirect to attempt.url.
```

Plain HTTP:

```typescript
import { createHash, randomBytes } from "node:crypto"

const codeVerifier = randomBytes(32).toString("base64url")
const codeChallenge = createHash("sha256").update(codeVerifier).digest("base64url")
const state = randomBytes(32).toString("base64url")

await storePendingAuthorization({ state, codeVerifier, userId, sessionId }) // server-side, single use

const url = new URL("https://link.ofauth.com/v3/authorize")
url.search = new URLSearchParams({
  response_type: "code",
  client_id: config.clientId,
  redirect_uri: config.redirectUri,
  state,
  code_challenge: codeChallenge,
  code_challenge_method: "S256",
  scope: "profile:read messages:read"
}).toString()

// Redirect the browser to url. Send nothing else to the client.
```

## Redirect URI rules

Developer integrations register redirect URIs per environment client in the dashboard, up to 20 per client. `redirect_uri` must match a registered value exactly (scheme, host, port, path, trailing slash).

| Rule | Sandbox and Test | Live |
| --- | --- | --- |
| HTTPS with a domain hostname | Allowed | Required |
| `http://127.0.0.1:<port>/...` or `http://[::1]:<port>/...` | Allowed | Rejected |
| `http://localhost` or any other `http://` host | Rejected | Rejected |
| IP address, `localhost`, or `*.localhost` over HTTPS | Rejected | Rejected |
| Fragment, wildcard, or userinfo | Rejected | Rejected |

Use the same host for the app and its callback so the initiating session cookie is available. Operator private apps have different callback rules; see [Operator private apps](#operator-private-apps).

## Browser surface

| Platform | Use |
| --- | --- |
| Web | The system browser or a top-level window (a normal redirect). |
| iOS | `ASWebAuthenticationSession` |
| Android | A Custom Tab |

No iframe, no `postMessage`. Native apps ask their backend for the URL, and the callback is the backend's HTTPS URL. After the callback, the backend can open a one-use Universal Link or App Link carrying only a random reference to the app's sign-in attempt. No custom URL schemes, no direct native callback, no token exchange from the native client.

What the user sees: OFAuth establishes the owner (guests can connect without an OFAuth signup), the owner links an OnlyFans account or chooses accounts already linked, approves the requested scopes, and returns to the callback. The consent screen may offer "Also share accounts I link later" (off by default, hidden for guests); new accounts then arrive via `authorization.updated` with `reason: "auto_added"` (see [webhooks.md](webhooks.md#lifecycle-events)).

## Callback

| Result | Query |
| --- | --- |
| Success | `code=<opaque>&state=<original>&iss=<issuer>` |
| Denied or cancelled | `error=access_denied&state=<original>` |
| Expired or invalid request | `error=invalid_request&state=<original>` |

If the request cannot be tied to a registered redirect URI, OFAuth shows the error itself instead of redirecting. The code expires after 5 minutes and is single use.

Accept the state only for the same app user and browser session that started the request; a mismatched session must not consume a legitimate pending attempt. Validate `iss` against `https://api.ofauth.com/v3/link`. The callback carries no account list.

```typescript
const params = new URL(request.url).searchParams
const state = params.get("state")
if (!state) throw new Error("Missing OAuth state")

// Atomically deletes and returns the record only if it matches this user and session and is unexpired.
const pending = await consumePendingAuthorization(state, session)
if (!pending) throw new Error("Invalid OAuth state")

const error = params.get("error")
if (error) return handleAuthorizationError(error) // access_denied, invalid_request

if (params.get("iss") !== "https://api.ofauth.com/v3/link") throw new Error("Unexpected issuer")
const code = params.get("code")
if (!code) throw new Error("Missing authorization code")

const tokens = await exchangeCode(code, pending.codeVerifier)
```

With the SDK: `const { code } = link.parseCallback(request.url, pending.state)` then `await link.exchangeCode({ code, codeVerifier: pending.codeVerifier })`.

## Token endpoint

| Requirement | Value |
| --- | --- |
| Client auth | Exactly one per request: `client_secret_basic` (HTTP Basic) or `client_secret_post` (`client_id` and `client_secret` form fields). Missing or combined methods are rejected. |
| Body | `application/x-www-form-urlencoded`. Not JSON. |
| Grants | `authorization_code`, `refresh_token`. Public clients, implicit, password, and `client_credentials` are unsupported. |
| `resource` | Not needed. Unsupported audiences return `invalid_target`. |

Code exchange fields: `grant_type=authorization_code`, `code`, `redirect_uri` (the exact value from the authorization request), `code_verifier`.

```typescript
const basic = Buffer.from(
  `${encodeURIComponent(config.clientId)}:${encodeURIComponent(config.clientSecret)}`
).toString("base64")

async function tokenRequest(form: Record<string, string>) {
  const response = await fetch("https://api.ofauth.com/v3/link/token", {
    method: "POST",
    headers: {
      authorization: `Basic ${basic}`,
      "content-type": "application/x-www-form-urlencoded"
    },
    body: new URLSearchParams(form)
  })
  const body = await response.json()
  if (!response.ok) throw new OAuthError(response.status, body.error, body.reason, body.recovery?.action)
  return body
}

const exchangeCode = (code: string, codeVerifier: string) =>
  tokenRequest({
    grant_type: "authorization_code",
    code,
    redirect_uri: config.redirectUri,
    code_verifier: codeVerifier
  })
```

## Token response

Sent with `Cache-Control: no-store`.

```json
{
  "access_token": "<opaque access token>",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "<opaque refresh token>",
  "scope": "identity:read profile:read",
  "authorization_id": "<authorization identifier>"
}
```

| Field | Notes |
| --- | --- |
| `access_token` | Opaque Bearer secret. Do not decode it. |
| `expires_in` | Access-token lifetime in seconds (initially 15 minutes). Compute expiry from it. |
| `refresh_token` | Returned on every successful code exchange. Stable, bound to the client. |
| `scope` | Granted scopes. May be fewer than requested; check it. |
| `authorization_id` | Optional informational extension. Use `GET /v3/authorization` when you need the authorization resource. |

Get account ids from `GET /v3/linked-accounts` (see [access.md](access.md#linked-accounts)), not from the token response.

## Refresh

Fields: `grant_type=refresh_token`, `refresh_token`, optional `scope` (a subset, to narrow the new access token; it cannot add permissions).

- Each success returns a new access token and the same refresh token.
- Independent processes can refresh concurrently; both access tokens stay valid. No distributed lock or replay journal is needed.
- Retry a refresh after a lost response or a failed save. Retry `server_error` and temporary network or storage failures with bounded backoff.
- Refresh tokens expire after 90 days without a successful refresh. Access calls do not extend that window. Revoked or expired credentials never revive. Owner disconnect, guest expiry, and client restrictions can end access earlier.
- `invalid_grant` requires a new authorization. Mark the user for re-authorization.

```typescript
async function getAccessToken(sessionId: string) {
  const stored = await loadSession(sessionId)
  if (stored.accessToken && stored.accessTokenExpiresAt > Date.now() + 60_000) return stored.accessToken
  try {
    const tokens = await tokenRequest({ grant_type: "refresh_token", refresh_token: stored.refreshToken })
    // Conditional write: skip if the user reconnected or disconnected meanwhile.
    await saveAccessToken(sessionId, stored.registrationVersion, {
      accessToken: tokens.access_token,
      accessTokenExpiresAt: Date.now() + tokens.expires_in * 1000
    })
    return tokens.access_token
  } catch (err) {
    if (err instanceof OAuthError && err.code === "invalid_grant") await markReauthorizationRequired(sessionId)
    throw err
  }
}
```

With the SDK: `await link.refresh({ refreshToken })`. Refresh when fewer than about 60 seconds remain, or when an Access call returns `recovery.action` `obtain_external_oauth_token`.

## Storage model

Suggested records (adapt to the codebase or let the app's OAuth framework own them):

- **Credential**: app user id, client ID (environment), encrypted refresh token, access token and expiry (server-side only), granted scopes, optional `authorization_id`, status (active or needs re-authorization), and a registration version for conditional writes so an older callback or refresh cannot overwrite a newer registration.
- **Linked account**: client ID (environment), linked account id, app user id, status (`connected`, `needs_reconnect`, `disconnected`), scopes, optional cached `username` and `display_name` (they can change; `provider_account_id` is stable). Store the user's selected account per client ID.

The docs' token-storage key example is `JSON.stringify([clientId, appUserId])`. If a refresh token is lost, send the user through authorization again.

## Requesting more scopes or accounts

Run authorization again with the additional scopes. Existing grants and accounts stay shared; the owner chooses which accounts receive the new scopes. The authorization keeps its identity and you receive `authorization.updated` with `reason: "consent_updated"`. The new code exchange issues an independent credential with the newly approved scopes; older credentials do not gain scopes. Re-read `GET /v3/linked-accounts` for the actual per-account scopes.

## Revoke and disconnect

`POST https://api.ofauth.com/v3/link/revoke`, either client auth method, form body: `token`, optional `token_type_hint` (`refresh_token` or `access_token`).

- Revoking a refresh token ends that credential and all its access tokens. Revoking an access token ends only that token.
- Returns `200`, also for unknown, already revoked, or foreign-client tokens.
- Revoke never ends the whole authorization or durable owner consent, and does not send `authorization.revoked`.

To end the whole authorization and all its credentials (for "Disconnect OnlyFans"), send `DELETE https://api.ofauth.com/v3/authorization` with a current Bearer token, then delete stored credentials. This sends `authorization.revoked` with `reason: "oauth_revocation"`. Repeating it is safe, but a later request with the dead token can return `invalid_token`.

To drop one account but keep the authorization, use `DELETE /v3/linked-accounts/{id}` (see [access.md](access.md#linked-accounts)).

## User info

Request `identity:read` to read the consenting person's OFAuth profile at `GET https://api.ofauth.com/v3/link/userinfo` with the Bearer token. Fields: `id` (equals `sub`), `sub` (stable person id; use this to identify them), `name`, `email` (only when known and permitted), `email_verified`, `picture`. Without identity consent it returns `403` with a `WWW-Authenticate` challenge. Identity consent can be granted without sharing an OnlyFans account.

## Discovery

`GET https://api.ofauth.com/.well-known/oauth-authorization-server/v3/link` returns RFC 8414 metadata: `issuer` `https://api.ofauth.com/v3/link`, `authorization_endpoint` `https://link.ofauth.com/v3/authorize`, `token_endpoint`, `revocation_endpoint`, `userinfo_endpoint`, `response_types_supported` `["code"]`, `grant_types_supported` `["authorization_code", "refresh_token"]`, `code_challenge_methods_supported` `["S256"]`, `token_endpoint_auth_methods_supported` and `revocation_endpoint_auth_methods_supported` `["client_secret_basic", "client_secret_post"]`, and `scopes_supported`.

This is plain OAuth: no OIDC discovery, no ID tokens, no public JWKS. Configure explicit endpoints in a generic OAuth library unless its RFC 8414 behavior is verified. For Better Auth, follow https://docs.ofauth.com/v3/sdk/better-auth.

## Operator private apps

Full guide: https://docs.ofauth.com/v3/guides/operator-apps.

Use this when the app acts on accounts in the user's own Operator workspace (a self-hosted app, script, or agent). Requests use the workspace's Operator allowance and the approving member's current rights. Developer integrations (dashboard client IDs per environment, Developer plan billing) are for apps that onboard other people's accounts.

**Register.** Discover `registration_endpoint` from the discovery document, then register each installation per RFC 7591:

```http
POST https://api.ofauth.com/v3/link/register
Content-Type: application/json

{
  "client_name": "My app",
  "redirect_uris": ["https://my-app.example/oauth/ofauth/callback"],
  "token_endpoint_auth_method": "client_secret_basic",
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "application_type": "web",
  "scope": "access:read connections:manage"
}
```

- `201` returns `client_id`, a unique `client_secret`, and the registered metadata. Store them privately and reuse them across restarts. Never embed a shared secret in a public app. A native app or agent may use `application_type: "native"` with its own loopback callback and per-installation credentials (RFC 8252 section 8.4).
- Registration alone grants nothing. The first accepted browser consent binds it to the approving workspace. An unused registration expires after 24 hours. The endpoint is rate-limited.
- Register one exact callback per installation: HTTPS, or loopback HTTP for local use (`127.0.0.1`, `localhost`, or `[::1]`). Wildcards, fragments, embedded credentials, and OAuth response parameters are rejected. Unknown metadata is ignored.
- Optional `ofauth_reconnect_return_uri`: a separate return URL accepted only for reconnect (useful when an auth library owns the OAuth callback). Invalid values return `invalid_client_metadata`.

**Authorize and tokens.** Same browser flow, issuer, token endpoint, form body, and client auth methods as above. Scripts use the browser flow too; there are no permanent API keys. Access tokens last 900 seconds; refresh tokens stay the same and expire after 90 days without a successful refresh. A token for the MCP resource does not grant API access.

| Scope | Access |
| --- | --- |
| `access:read` | Read supported Access account data. |
| `access:write` | Perform supported account changes. |
| `access:destructive` | Perform supported destructive actions. Private apps get no extra confirmation step, so confirm with the user in your own app first. |
| `connections:manage` | Start provider reconnect for a granted account. |

**Call Access.** List accounts with `GET /v3/linked-accounts`, then send one id as `x-linked-account-id` on the usual `/v3/access/*` paths. There is no operation-listing endpoint: take each operation's method, path, and scope from the V3 API reference (https://docs.ofauth.com/v3/api-reference, indexed in https://docs.ofauth.com/llms.txt). A path that is not available to private apps returns `403 operation_unavailable`; purchases, batch users, multi-list writes, and generic settings are unavailable. Operator Essential permits reads; Pro and Max also permit supported writes.

Private-app requests run directly, with no browser step: there is no recent-authentication requirement, no confirmation URL, and no confirmation header. Behaviour to handle:

- Every write or destructive request needs an `Idempotency-Key` header (a fresh random value per logical action). Without it the request returns `403 idempotency_conflict` with `recovery.action` `correct_request`. Reuse a key only to retry the identical request.
- Idempotency keys are honoured for 90 days after their first use. Retrying a key whose write already succeeded sends nothing again: it returns `200` with `{ "outcome": "already_succeeded", "operation_id" }` and the header `Idempotent-Replayed: true`. The original response body is not stored, so read the result back if you need it. A key whose write failed is spent: try again with a new key. The same key with a different request returns `idempotency_conflict`.
- If OFAuth cannot tell whether a write reached OnlyFans, it returns `409 unknown_outcome` with `recovery.action` `do_not_repeat` and an `operation_id`, and it keeps returning that for the same key. Check the account (for example, read the post or message back) before deciding to send it again with a new key.
- A missing `x-linked-account-id` returns `400 linked_account_required`.
- Errors use `{ "error", "recovery": { "action" }, "requestId" }` and set `x-request-id`; log `requestId`.

Browser confirmation (`confirmation_url`, `X-OFAuth-Confirmation`) belongs to OFAuth's MCP server and the dashboard only. Do not build it into a private app.

**Reconnect, revoke, disconnect.** `POST /v3/linked-accounts/{id}/reconnect` (see [access.md](access.md#reconnect)) with `redirect_uri` set to the registered callback or `ofauth_reconnect_return_uri`. Revoke and `DELETE /v3/authorization` work as above. Operator private apps do not include Realtime, Vault+, events, or webhooks.
