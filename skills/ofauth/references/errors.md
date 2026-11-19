# Errors and recovery

Full reference: https://docs.ofauth.com/v3/api-reference/errors and https://docs.ofauth.com/v3/reference/error-handling.

V3 has two error shapes. `POST /v3/link/token` and `POST /v3/link/revoke` follow RFC 6749. Everything under `/v3/access`, `/v3/vault-plus`, and `/v3/linked-accounts` uses the API error body.

## API error body

```json
{
  "error": "linked_account_reconnect_required",
  "recovery": { "action": "reconnect_linked_account" },
  "requestId": "4f5c3f0e-8d7c-4ff5-9b7f-7d0d9e7c2a61"
}
```

Branch on `recovery.action`. Log the operation, HTTP status, `error`, `recovery.action`, and `requestId` (also sent as `x-request-id`); quote `requestId` in support requests. Never log authorization headers, tokens, secrets, or OnlyFans session data. SDKs expose these as `code`, `recoveryAction` (TypeScript) or `recovery_action` (Python), and `requestId` / `request_id` on `OFAuthAPIError`.

A `401` does not always mean the OnlyFans account needs a new sign-in: the token may be invalid, the owner may have ended access, or the account may need a reconnect. Read the error.

## API error codes

| HTTP | `error` | `recovery.action` | Cause |
| --- | --- | --- | --- |
| 401 | `external_oauth_bearer_required` | `obtain_external_oauth_token` | No bearer, or an API key header was sent. |
| 401 | `invalid_token` | `obtain_external_oauth_token` | Access token expired, malformed, or not valid here. Refresh and retry. |
| 401 | `grant_revoked_or_expired` | `restart_authorization` | The permission behind this token has ended. |
| 401 | `authorization_required` | `restart_authorization` | The authorization is not usable. |
| 401 | `linked_account_reconnect_required` | `reconnect_linked_account` | The account's OnlyFans session ended. The owner must reconnect. |
| 403 | `linked_account_required` | `select_linked_account` | Missing or malformed `x-linked-account-id`. |
| 403 | `linked_account_not_granted` | `grant_linked_account` | The account is not part of this authorization. |
| 404 | `linked_account_not_found` | `list_linked_accounts` | Path account not in this authorization (`/v3/linked-accounts/{id}`). |
| 403 | `linked_account_vault_plus_disabled` | `enable_linked_account_vault_plus` | Vault+ off for the account. |
| 403 | `vault_plus_disabled` | `enable_vault_plus` | Vault+ off for the environment. |
| 403 | `missing_scope` | `request_additional_consent` | Token lacks the operation's scope. |
| 403 | `additional_consent_required` | `request_additional_consent` | Scope in the token but not approved for this account. |
| 403 | `integration_not_approved` | `complete_integration_approval` | Live not approved. |
| 403 | `subscription_required` | `update_developer_plan` | No active Developer plan. |
| 403 | `integration_suspended` | `contact_integration_owner` | Integration or a required feature unavailable. |
| 403 | `security_suspended` | `contact_integration_owner` | Suspended for security reasons. |
| 403 | `client_deleted` | `contact_integration_owner` | The environment client was deleted. |
| 409 | `linked_account_settings_conflict` | `retry` | Concurrent settings change. |
| 409 | `linked_account_settings_fixed` | (none) | Sandbox settings cannot change. |
| 429 | `quota_exhausted` | `retry_after_reset` | Test daily quota used. `Retry-After` gives seconds to the UTC reset. |
| 429 | `rate_limited` | `retry` | Too many requests. Wait `Retry-After`. |
| 503 | `authorization_temporarily_unavailable` | `retry` | OFAuth could not validate the request now. Back off. |

Body and query validation errors return `400` with the same body and a descriptive `error` such as `invalid_request`. The reconnect route adds `400 invalid_redirect_uri`, `409 linked_account_reconnect_not_required`, and `409 linked_account_reconnect_unavailable` (see [access.md](access.md#reconnect)).

When OnlyFans rejects an admitted request: `provider_request_rejected` (`do_not_repeat` or `check_request`), `provider_resource_not_found` (`check_resource`), `linked_account_reconnect_required` (`reconnect_linked_account`), `rate_limited` (`retry`), or `transient_provider_failure` (`retry`).

## Recovery actions

| `recovery.action` | Implementation |
| --- | --- |
| `obtain_external_oauth_token` | Refresh the access token and retry once. If refresh fails with `invalid_grant`, start authorization. |
| `restart_authorization` | Delete the stored refresh token and send the user through authorization. |
| `reconnect_linked_account` | Pause that account and start a [reconnect](access.md#reconnect). The id stays the same; resume when it is `connected` again. Refresh cannot fix this. |
| `select_linked_account` | Send a valid `x-linked-account-id`. A code bug; fix it. |
| `grant_linked_account` | Run authorization so the owner can share this account. |
| `list_linked_accounts` | Refresh the account list from `GET /v3/linked-accounts`. |
| `request_additional_consent` | Run authorization with the required scopes. |
| `enable_linked_account_vault_plus` | `PATCH /v3/linked-accounts/{id}/settings` with `{ "vault_plus": { "enabled": true } }`. |
| `enable_vault_plus` | The user turns Vault+ on for the environment in the dashboard. |
| `retry_after_reset` | Wait `Retry-After` seconds. |
| `retry` | Wait `Retry-After` when present, else back off longer after each attempt, with jitter and a cap. |
| `complete_integration_approval` | The user requests Live approval in the dashboard. |
| `update_developer_plan` | The user activates the Developer plan. |
| `contact_integration_owner` | Not recoverable in code. Notify the integration owner. |
| `do_not_repeat` | OnlyFans refused. Do not retry unchanged. |
| `check_request` | Fix the request before retrying. |
| `check_resource` | The OnlyFans resource does not exist. |

Retry only when the recovery action allows it. For writes (messages, posts, stories), a network error does not prove failure: check whether the item was created before sending again, and never automatically repeat a write whose outcome is unknown.

## OAuth endpoint errors

```json
{
  "error": "invalid_grant",
  "error_description": "request rejected",
  "reason": "developer_plan_unavailable",
  "recovery": { "action": "contact_integration_owner" }
}
```

| HTTP | `error` | Cause | Do |
| --- | --- | --- | --- |
| 400 | `invalid_request` | Missing, repeated, unsupported, or malformed field; not form-encoded; ambiguous client authentication. | Fix the request. |
| 400 | `unsupported_grant_type` | Grant not `authorization_code` or `refresh_token`. | Fix the request. |
| 401 | `invalid_client` | Missing or invalid client authentication, or client unavailable. | Check credentials and environment. Do not loop: 10 failures in 10 minutes block the client ID. |
| 400 | `invalid_grant` | Code or refresh token expired, used, or revoked; PKCE or redirect URI mismatch. | Start a new authorization. |
| 400 | `invalid_scope` | Unknown scope or a permission expansion. | Fix the scope list. |
| 400 | `invalid_target` | `resource` is not `https://api.ofauth.com/v3`, or the client is in the wrong environment for the approval. | Drop `resource`; use the right client. |
| 429 | `rate_limited` | Too many requests or failed client authentications. | Wait `Retry-After`. |
| 500 | `server_error` | Temporary server failure. | Retry with bounded backoff. Refresh can be retried with the same refresh token. |

`reason` and `recovery` appear only when the integration is deleted, its plan is unavailable, or it is suspended (`client_deleted`, `developer_plan_unavailable`, `security_suspended`). Surface these to the integration owner. After routine client-secret rotation, OAuth endpoints reject the retired secret; existing customer tokens do not end.

## Callback errors

The redirect can carry `error=access_denied` (user declined or cancelled; in Sandbox also a refused selection, with `error_description`) or `error=invalid_request` (request expired or malformed), both with the original `state`. Consume the state, then offer a retry. Errors that cannot be tied to a registered redirect URI are shown on the OFAuth page instead.

## Handling pattern

```typescript
async function callWithRecovery<T>(sessionId: string, linkedAccountId: string, call: () => Promise<Response>, isRead: boolean): Promise<T> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await call()
    if (response.ok) return response.json() as Promise<T>

    const body = await response.json()
    log.warn("ofauth error", { status: response.status, code: body.error, action: body.recovery?.action, requestId: body.requestId })

    switch (body.recovery?.action) {
      case "obtain_external_oauth_token":
        if (attempt > 0) break
        await forceRefresh(sessionId) // see oauth.md; invalid_grant marks re-authorization
        continue
      case "retry":
      case "retry_after_reset":
        if (!isRead && body.recovery.action === "retry") break // writes: the caller checks whether it completed before resending
        await sleep(retryDelay(response.headers, attempt))
        continue
      case "reconnect_linked_account":
        await markNeedsReconnect(linkedAccountId)
        break
      case "list_linked_accounts":
        await syncLinkedAccounts(sessionId)
        break
      case "restart_authorization":
      case "grant_linked_account":
      case "request_additional_consent":
        await markReauthorizationRequired(sessionId)
        break
    }
    throw new OFAuthError(response.status, body.error, body.recovery?.action, body.requestId)
  }
  throw new Error("OFAuth retries exhausted")
}
```

`retryDelay` should use `Retry-After` when present, else exponential backoff with jitter. For `retry_after_reset` with a long wait, schedule a job instead of sleeping in a request.