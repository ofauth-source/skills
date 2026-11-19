# Access, scopes, and linked accounts

Full reference: https://docs.ofauth.com/v3/api-reference, https://docs.ofauth.com/v3/api-reference/authentication, and https://docs.ofauth.com/v3/api-reference/account/linked-accounts. Each operation's own reference page shows its method, path, input fields, response, and required scope.

## Contents

- [Headers](#headers)
- [Calling an operation](#calling-an-operation)
- [Scopes](#scopes)
- [Linked accounts](#linked-accounts)
- [Linked account status](#linked-account-status)
- [Reconnect](#reconnect)
- [Pagination](#pagination)
- [Rate limits and quotas](#rate-limits-and-quotas)
- [Media](#media)

## Headers

| Header | Value |
| --- | --- |
| `Authorization` | `Bearer <access_token>`. Required on Access, Vault+, and linked-account requests. |
| `x-linked-account-id` | A linked account id of this token's authorization. Required on Access and Vault+ requests. Linked-account routes take the id in the URL instead. |
| `Content-Type` | `application/json` for JSON request bodies. Upload URLs take file bytes; use the content type the operation lists. |

The account id selects an account; the token grants access. An account id alone authorizes nothing. API keys are rejected: an API key header or a missing bearer returns `401 external_oauth_bearer_required`. Every response carries `x-request-id`; log it.

The access-token APIs send no CORS headers. Call them from the backend only, never from a browser.

## Calling an operation

Prefer the SDK; it sends both headers, paginates, and throws `OFAuthAPIError` with `code`, `recoveryAction`, and `requestId`.

```typescript
import { OFAuthClient } from "@ofauth/onlyfans-sdk"

const client = new OFAuthClient({
  accessToken: async () => getAccessToken(sessionId), // the app owns storage and refresh
  linkedAccountId
})
const profile = await client.access.profile.get()
```

Plain HTTP:

```typescript
async function access(sessionId: string, linkedAccountId: string, path: string, init: RequestInit = {}) {
  const accessToken = await getAccessToken(sessionId) // refresh when < 60s left; see oauth.md
  const response = await fetch(`https://api.ofauth.com/v3/access${path}`, {
    ...init,
    redirect: "error", // never follow redirects with a bearer token
    headers: {
      authorization: `Bearer ${accessToken}`,
      "x-linked-account-id": linkedAccountId,
      "content-type": "application/json",
      ...init.headers
    }
  })
  if (!response.ok) {
    const body = await response.json()
    throw new AccessError(response.status, body.error, body.recovery?.action, body.requestId)
  }
  return response.json()
}

const profile = await access(sessionId, linkedAccountId, "/profile")
```

Only use operation paths and fields you have confirmed on the operation's reference page (find them through https://docs.ofauth.com/llms.txt). Send only the input fields listed for the operation. If the user needs an operation you cannot find, ask them or tell them to contact support@ofauth.com. Do not guess paths.

Writes: if a network error leaves the outcome unknown, check whether the post, message, or story was created before sending again. Do not assume the API detects repeated writes. (Operator private apps send `Idempotency-Key`; see [oauth.md](oauth.md#operator-private-apps).)

## Scopes

Request only the scopes the app uses; the owner sees each one. Read and write are separate: write does not include read. `GET https://api.ofauth.com/.well-known/oauth-authorization-server/v3/link` lists every supported scope in `scopes_supported`. Operator private apps use different scopes; see [oauth.md](oauth.md#operator-private-apps).

| Scope | Grants |
| --- | --- |
| `identity:read` | Read the consenting person's OFAuth name, email, and profile image (User info). No OnlyFans account action. |
| `profile:read` | Read the profile, including the OnlyFans account ID, counts, links, settings, and release forms. |
| `profile:write` | Update the profile, links, social buttons, settings, release forms. |
| `notifications:read` | Read notifications and unread counts. |
| `notifications:write` | Mark notifications as read. |
| `posts:read` | Read posts, comments, buyers, tips, bookmarks. |
| `posts:write` | Create, edit, delete, pin, archive posts; manage comments and bookmarks. |
| `stories:read` | Read stories, highlights, viewers, stats. |
| `stories:write` | Create and delete stories and highlights; like, watch, block. |
| `queue:read` | Read scheduled posts, messages, and saved-for-later items. |
| `queue:write` | Publish or delete queued items. |
| `messages:read` | Read chats, messages, chat media, mass messages, templates. |
| `messages:write` | Send, delete, pin, like messages; mute or hide chats; send mass messages. |
| `subscribers:read` | Read subscribers, activity, awards, subscription history. |
| `subscribers:write` | Set notes, display names, and discounts for subscribers. |
| `subscriptions:read` | Read the accounts this user subscribes to. |
| `subscriptions:write` | Subscribe and hide or show posts from subscriptions. |
| `users:read` | Look up users, search, friends, blocked and restricted lists. |
| `users:write` | Block, unblock, restrict, unrestrict users. |
| `lists:read` | Read user lists, fans and following lists, labels. |
| `lists:write` | Create, edit, sort, delete lists; add or remove members; manage labels. |
| `promotions:read` | Read promotions, bundles, offers, tracking links, trial links. |
| `promotions:write` | Create, end, delete promotions, bundles, tracking links, trial links. |
| `analytics:read` | Read profile, post, story, stream, message, promotion, and traffic analytics. |
| `finances:read` | Read earnings, transactions, chargebacks, payouts, referrals. |
| `vault:read` | Read vault media and vault lists. |
| `vault:write` | Upload media; create, edit, delete vault lists; hide media. |

`403 missing_scope`: the token lacks the operation's scope. `403 additional_consent_required`: the selected account has not approved it. Both mean: run authorization again with the needed scopes ([oauth.md](oauth.md#requesting-more-scopes-or-accounts)).

## Linked accounts

A linked account is one OnlyFans account. It belongs to its owner, not to the app; the owner can share it with several apps and revoke access at any time. Every Access call targets exactly one linked account.

Ids come from `GET /v3/linked-accounts` and from lifecycle webhooks. The callback is not an account list. The same OnlyFans account has different ids in Sandbox, Test, and Live.

All routes take the bearer token and only see accounts of that token's authorization. No route lists accounts across authorizations.

| Route | Purpose |
| --- | --- |
| `GET /v3/linked-accounts?limit=&cursor=` | List, ordered by `id`. `limit` 1 to 100, default 50. Bad `limit` or `cursor` returns `400 invalid_request`. |
| `GET /v3/linked-accounts/{id}` | One account. Not in this authorization: `404 linked_account_not_found`. |
| `DELETE /v3/linked-accounts/{id}` | Remove this authorization's access to one account. Returns `204`; repeating succeeds. Sends `linked_account.disconnected` with `reason: "developer_deleted"`. |
| `POST /v3/linked-accounts/{id}/reconnect` | Create an owner reconnect journey. See [Reconnect](#reconnect). |
| `GET` / `PATCH /v3/linked-accounts/{id}/settings` | Per-account settings (Vault+). See [vault-plus.md](vault-plus.md#per-account-setting). |
| `GET /v3/authorization` | The current authorization: identity, effective scope, and a first page of linked accounts. |

SDK: `client.linkedAccounts.iterate()`, `.get({ id })`, `.remove({ id })`, `.createReconnect({ id, body })`.

List response:

```json
{
  "data": [
    {
      "id": "conn_example123",
      "provider": "onlyfans",
      "provider_account_id": "123456789",
      "username": "creator",
      "display_name": "Creator Name",
      "status": "connected",
      "scopes": ["profile:read", "messages:read"],
      "granted_at": "2026-10-01T12:00:00.000Z"
    }
  ],
  "has_more": false,
  "next_cursor": null
}
```

`provider_account_id` is the stable OnlyFans user id. `username` and `display_name` can change or be `null`. In chat routes, the fan's OnlyFans user id is a different thing from the linked account id that selects the creator.

One authorization can hold several accounts (for example an agency). Let the user pick which account to operate; the same access token works for all of them, only the header changes. Keep the list current with lifecycle webhooks ([webhooks.md](webhooks.md#lifecycle-events)).

Header errors: missing or malformed `x-linked-account-id` returns `403 linked_account_required`; an account outside the authorization returns `403 linked_account_not_granted`.

## Linked account status

| Status | Meaning | What to do |
| --- | --- | --- |
| `connected` | Usable. | Call Access normally. |
| `needs_reconnect` | The OnlyFans session is no longer usable. Your access remains. | Pause that account only. Calls return `linked_account_reconnect_required`. Start a [reconnect](#reconnect). The id stays the same. |
| `disconnected` | The owner disconnected the account. | Stop calling Access for it. |

Refreshing an OAuth token never fixes `needs_reconnect`.

## Reconnect

Guide: https://docs.ofauth.com/v3/guides/reconnect.

```http
POST https://api.ofauth.com/v3/linked-accounts/{id}/reconnect
Authorization: Bearer <access_token>
Content-Type: application/json

{ "redirect_uri": "https://app.example.com/ofauth/reconnected", "state": "<random, one use>" }
```

- `redirect_uri` must exactly match a registered redirect URI (Developer) or the registered callback or `ofauth_reconnect_return_uri` (Operator private app). If an auth library owns the OAuth callback, register a separate return URL for reconnect. Unregistered: `400 invalid_redirect_uri`.
- `201` returns `url`, `expires_at` (the journey lasts 15 minutes), `authorization_id`, `linked_account_id`. Send the owner's browser to `url`; do not build a Link URL yourself.
- `409 linked_account_reconnect_not_required` (already connected) or `409 linked_account_reconnect_unavailable`.
- The owner returns with `state` and `reconnect=completed`, `canceled`, `expired`, or `denied`. Bind `state` to the app session and target account server-side and consume it once. Read the target account from the saved state, never from the query.
- After return, or on `linked_account.reconnected`, call `GET /v3/linked-accounts/{id}` and resume only when `status` is `connected`. Reconnect keeps the authorization, account id, and scopes. It does not issue tokens or tell you whether an interrupted write completed.

## Pagination

| List | Items | Continue while | Next request |
| --- | --- | --- | --- |
| Access lists with a cursor | `items` | `page.hasMore` is `true` | Send `page.nextCursor` as `cursor`. |
| Linked accounts | `data` | `has_more` is `true` | Send `next_cursor` as `cursor`. |
| Vault+, analytics, others | See the endpoint | See the endpoint | Follow that endpoint's reference. |

Copy cursors unchanged and keep the same filters and sort. Profile reads and create requests return the object directly. SDK lists have `iterate()`.

## Rate limits and quotas

Reference: https://docs.ofauth.com/v3/reference/rate-limiting.

| Request type | Shared by | Limit |
| --- | --- | --- |
| Token and revoke requests | Client IP address | 120 requests, restored steadily over 60 seconds. |
| Failed client authentication | Client ID | After 10 failures in 10 minutes, blocked until the window ends. |
| Linked-account and Access requests | Authorization and environment | 600 requests, restored steadily over 60 seconds. Accounts in one authorization share it. |

Responses carry `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, and on rejection `Retry-After`. Over the limit: `429 rate_limited`, `recovery.action` `retry`. Wait `Retry-After` plus a small random delay. `RateLimit-Reset` is when all requests are restored; it does not mean retry sooner.

Test has a separate daily Access operation quota: `429 quota_exhausted`, `recovery.action` `retry_after_reset`, `Retry-After` set to seconds until the UTC reset.

## Media

Media URLs in responses expire. Read the post, message, or Vault item again for a fresh URL, and save media ids, not URLs. Guides: https://docs.ofauth.com/v3/guides/media (Vault browsing and lists), https://docs.ofauth.com/v3/guides/media-items (attaching media), https://docs.ofauth.com/v3/guides/display-media, and https://docs.ofauth.com/v3/guides/drm-playback for protected video. For stored copies findable by media id, use [Vault+](vault-plus.md).
