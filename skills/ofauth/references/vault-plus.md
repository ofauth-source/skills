# Vault+

Full reference: https://docs.ofauth.com/v3/guides/media-storage and the Vault+ operation pages under https://docs.ofauth.com/v3/api-reference (for example https://docs.ofauth.com/v3/api-reference/vault+/query-vault+-media).

Vault+ is an optional feature for Developer integrations (not Operator private apps). It stores OnlyFans Vault media so the app can load files and thumbnails faster and find them by OnlyFans media id, even when they are not attached to a post, message, or story, without handling expiring OnlyFans URLs. It works only for media you have added or synced. It lives under `https://api.ofauth.com/v3/vault-plus` and uses the same authentication as Access.

Do not confuse it with the OnlyFans Vault operations under `/v3/access/vault/*` (browse media, manage Vault lists), which are ordinary Access operations.

## Headers and scopes

| Header | Value |
| --- | --- |
| `Authorization` | `Bearer <access_token>` |
| `x-linked-account-id` | A linked account id of this authorization. Required on every route. |

| Scope | Routes |
| --- | --- |
| `vault:read` | All `GET` routes and `POST /v3/vault-plus/media/batch`. |
| `vault:write` | `POST /lists/:listId/add`, `PUT /lists/:listId/sync`, both `DELETE` routes, and the per-account settings `PATCH`. |

Request these in the [authorization request](oauth.md#authorization-request).

## Turning it on

| Level | How | Error when off |
| --- | --- | --- |
| Environment | The integration's Vault+ setting for the environment, in the dashboard. The user does this. | `403 vault_plus_disabled` (`recovery.action` `enable_vault_plus`). |
| Linked account | `PATCH /v3/linked-accounts/{id}/settings` (see below). | `403 linked_account_vault_plus_disabled` on `add` and `sync` (`enable_linked_account_vault_plus`). |

### Per-account setting

```http
GET   https://api.ofauth.com/v3/linked-accounts/{id}/settings
PATCH https://api.ofauth.com/v3/linked-accounts/{id}/settings
Authorization: Bearer <access_token>
Content-Type: application/json

{ "vault_plus": { "enabled": true } }
```

Response: `{ "linked_account_id": "conn_example123", "vault_plus": { "enabled": true } }`.

- `PATCH` accepts `vault_plus.enabled`, `vault_plus.overrides`, or both; send only what changes. Set an override field to `null` to fall back to the integration setting, or `"overrides": null` to clear all overrides. Supported override fields are listed at https://docs.ofauth.com/v3/api-reference/linked-accounts/update-linked-account-settings; do not guess others.
- Needs `vault:write` on the account, else `403 missing_scope`.
- Test accounts need a paid Developer plan, else `403 subscription_required`.
- Sandbox accounts always have Vault+ on; `PATCH` there returns `409 linked_account_settings_fixed`. Skip the call in Sandbox.
- A concurrent change returns `409 linked_account_settings_conflict` with `recovery.action` `retry`.

SDK: `client.linkedAccounts.settings.update({ id, body: { vault_plus: { enabled: true } } })`.

## Media ids

Routes use the raw OnlyFans media id. Items are grouped by that id, with cached variants keyed by label (such as `full`, `thumb`, `720`). Use `thumb` for thumbnails.

## Routes

| Method | Route | Purpose | SDK |
| --- | --- | --- | --- |
| `GET` | `/v3/vault-plus/lists` | Cached vault-list summaries for the account. | |
| `GET` | `/v3/vault-plus/lists/:listId/media` | Cached media membership for one list. | |
| `POST` | `/v3/vault-plus/lists/:listId/add` | Save missing media from the current list. Never removes. | `vaultPlus.lists.add({ listId })` |
| `PUT` | `/v3/vault-plus/lists/:listId/sync` | Make saved copies match the current list. | `vaultPlus.lists.sync({ listId })` |
| `GET` | `/v3/vault-plus/media` | Query saved media. | `vaultPlus.media.list({...})` |
| `GET` | `/v3/vault-plus/media/:mediaId` | One item with all variants. | `vaultPlus.media.get({ mediaId })` |
| `POST` | `/v3/vault-plus/media/batch` | Several items. Body `{ "mediaIds": [...] }` (1 to 100). Unknown ids are omitted. | `vaultPlus.media.batch({ body })` |
| `DELETE` | `/v3/vault-plus/media/:mediaId` | Delete one saved item. | `vaultPlus.media.delete({ mediaId })` |
| `DELETE` | `/v3/vault-plus/media` | Delete all saved media for the account. | `vaultPlus.media.deleteAll()` |
| `GET` | `/v3/vault-plus/status` | Storage status for the selected account. | |
| `GET` | `/v3/vault-plus/stats` | Totals for all accounts shared with the integration in this environment. Keep separate from `/status`. | |

`GET /v3/vault-plus/media` query parameters: `status` (`edge_only`, `pending`, `storing`, `stored`, `removed`), `type` (`image`, `video`, `audio`), `contentType`, `durationMin`, `durationMax`, `createdAtFrom`, `createdAtTo`, `cachedAtFrom`, `cachedAtTo`, `limit` (1 to 100), `cursor`, `sort` (`createdAt_desc`, `createdAt_asc`, `cachedAt_desc`, `cachedAt_asc`, `duration_desc`, `duration_asc`).

Example sync response:

```json
{ "synced": true, "listId": "123", "seen": 42, "addedItems": 2, "updatedItems": 1,
  "removedItems": 4, "unchangedItems": 35, "adoptedItems": 0 }
```

`adoptedItems` counts already saved files that this list's sync now tracks for the first time. Other response shapes are on each operation's reference page. Do not guess fields you have not seen there. Vault+ pagination differs from Access lists; follow the endpoint's response fields.

## Add and sync semantics

- `add` saves missing files and keeps all existing copies. It does not change what a later sync can remove.
- The first `sync` creates a baseline and removes nothing.
- Later syncs remove only files that were in the list at the last sync and have since left it.
- A file another list of the same account still needs is kept.
- Changed files are updated automatically.
- Check `/status` or `status=stored` before relying on a file being stored.
- Deleting saved copies never deletes the original OnlyFans content.

Use `add` when the app must never lose media; use `sync` when saved copies should mirror the list. For protected video, see https://docs.ofauth.com/v3/guides/vault-plus-drm.

Errors use the API error body; see [errors.md](errors.md).
