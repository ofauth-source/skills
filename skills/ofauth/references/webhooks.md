# Webhooks and Realtime

Full reference: https://docs.ofauth.com/v3/api-reference/webhooks, https://docs.ofauth.com/v3/api-reference/system-webhook-events, and https://docs.ofauth.com/v3/guides/realtime-onlyfans-events.

Webhooks, Realtime, and Vault+ are Developer integration features. Operator private apps do not receive them.

## Contents

- [Destinations](#destinations)
- [Verification](#verification)
- [Envelope](#envelope)
- [Lifecycle events](#lifecycle-events)
- [Realtime events](#realtime-events)
- [Realtime over Server-Sent Events](#realtime-over-server-sent-events)
- [Receiver rules](#receiver-rules)

## Destinations

The user configures destinations in the dashboard (External Integration, then Sandbox, Test, or Live, then **Webhooks**, **Create webhook**). You cannot create them in code.

| Setting | Notes |
| --- | --- |
| Environment | Sandbox, Test, or Live. Environments never share destinations or events. |
| Endpoint URL | Public HTTPS. For local development, the docs use a Cloudflare Quick Tunnel. |
| Send Realtime events | OnlyFans activity events for accounts shared with the integration. |
| Send lifecycle events | Account and access changes. Lifecycle deliveries are not billed. |
| Signing secret | Per destination, shown once. Rotatable. |
| Verify | The destination stays **Pending** until the receiver answers the signed challenge. It must be **Verified** and **Active** to receive events. |

Read the secret from an env var per environment, for example `OFAUTH_SANDBOX_WEBHOOK_SECRET`. Use placeholders in example files.

## Verification

Deliveries use Standard Webhooks (https://www.standardwebhooks.com): headers `webhook-id`, `webhook-timestamp`, `webhook-signature`. Verify the unmodified raw body with the destination's secret before reading any field. Use a Standard Webhooks library (the docs' example uses the `standardwebhooks` npm package), which checks both signature and timestamp. Register the raw-body handler before any JSON parser.

**Destination verification challenge.** Before normal delivery, OFAuth sends a signed JSON body with `type: "ofauth.webhook.challenge"` and a `challenge` string. It is not an event envelope and has no `eventId`. Verify its signature, then respond `2xx` with the exact `challenge` string as a `text/plain` body: no JSON quotes, no trailing newline.

```typescript
import { Webhook } from "standardwebhooks"

const verifier = new Webhook(process.env.OFAUTH_SANDBOX_WEBHOOK_SECRET!)

export async function handleOFAuthWebhook(rawBody: string, headers: Record<string, string>) {
  let event: any
  try {
    event = verifier.verify(rawBody, headers) // throws on bad signature or stale timestamp
  } catch {
    return new Response(null, { status: 400 })
  }

  if (event.type === "ofauth.webhook.challenge") {
    return new Response(event.challenge, { status: 200, headers: { "content-type": "text/plain" } })
  }

  if (event.environment !== config.environment) {
    log.warn("ofauth webhook for another environment", { eventId: event.eventId, type: event.type })
    return new Response(null, { status: 204 }) // never apply it to this environment's records
  }
  // Transactional outbox: record the event and its processing job in one transaction.
  // Both inserts ignore duplicates, so a redelivery after a crash still leaves exactly one
  // event row and one job row, and acknowledging can never drop an unscheduled event.
  await db.transaction(async (tx) => {
    await tx.insertEventIfNew(event.eventId, event)  // unique on eventId
    await tx.insertJobIfNew(event.eventId)           // unique on eventId; status "pending"
  })                                                 // throws -> 500 -> OFAuth retries
  return new Response(null, { status: 204 })
}

// A separate worker claims "pending" jobs, applies the event, then marks the job "done".
// A job that fails stays pending (or returns to pending after a lease timeout) and is retried.
```

If the transaction fails, return an error so OFAuth retries. Return success for verified event types you do not use. Do not record the event in one write and schedule its processing in a separate, later write: if the second write fails, the retry sees a duplicate event and the work is lost.

## Envelope

```json
{
  "eventId": "...",
  "type": "linked_account.expired",
  "schemaVersion": "1",
  "order": 42,
  "occurredAt": "2026-10-12T14:30:15.143Z",
  "environment": "live",
  "integrationId": "int_example123",
  "externalOauthClientId": "...",
  "authorizationId": "...",
  "linkedAccountId": "conn_example123",
  "test": false,
  "data": {
    "authorization_id": "...",
    "linked_account_id": "conn_example123",
    "reason": "provider_session_expired"
  }
}
```

| Field | Notes |
| --- | --- |
| `eventId` | Unique per event. Deduplicate on it. |
| `schemaVersion` | `"1"`. |
| `order` | For the same linked account, higher is newer. Do not compare across accounts. |
| `environment` | `sandbox`, `test`, or `live`. |
| `externalOauthClientId` | The environment client the authorization belongs to. |
| `authorizationId` | Map it to your user. |
| `linkedAccountId` | `null` for `authorization.*` events. |
| `test` | Marks test deliveries. Lifecycle events in Sandbox and Test also have `test: true`, including Test events about real accounts. |

`test: true` alone does not mean the ids are made up. Keep sample test data (made-up ids) out of customer records, and use `environment` to keep Sandbox, Test, and Live state separate.

## Lifecycle events

Delivered only by webhook. One action sends one event.

| Type | When | `data.reason` | Handling |
| --- | --- | --- | --- |
| `authorization.granted` | New authorization created. | `authorized` | Store account ids under the authorization. |
| `authorization.updated` | Re-consent changed accounts or scopes, or auto-add shared a new account. | `consent_updated`, `auto_added` | Add and remove accounts. When `scopes_changed` is `true`, re-read scopes from `GET /v3/linked-accounts`. |
| `authorization.revoked` | Whole authorization ended. | `holder_revoked` (owner disconnected the app), `oauth_revocation` (app called `DELETE /v3/authorization`), `integration_deleted`, `integration_suspended` | Delete stored refresh tokens. The user must authorize again. |
| `linked_account.expired` | One account needs a reconnect; access remains. | `provider_session_expired` | Pause operations for the account. |
| `linked_account.reconnected` | The account works again. | `reconnected` | Read `GET /v3/linked-accounts/{id}`; resume when `connected`. |
| `linked_account.disconnected` | The account no longer works for this authorization. | `owner_removed`, `owner_disconnected`, `owner_deleted`, `developer_deleted` | Stop operations and remove it from your records. |

`linked_account.*` events are sent to every authorization that includes the account. Revoking or expiring a single credential does not send `authorization.revoked`.

Payload fields (ids only; fetch names, status, and scopes from `GET /v3/linked-accounts`):

| Type | `data` fields |
| --- | --- |
| `authorization.granted` | `authorization_id`, `reason`, `linked_account_ids` |
| `authorization.updated` | `authorization_id`, `reason`, `scopes_changed`, `added_linked_account_ids`, `removed_linked_account_ids` |
| `authorization.revoked` | `authorization_id`, `reason`, `linked_account_ids` |
| `linked_account.*` | `authorization_id`, `linked_account_id`, `reason` |

A payload lists at most 100 ids. Above that, the lists are replaced by `linked_account_ids_truncated: true` and `linked_account_count`, plus `linked_accounts_url: "/v3/linked-accounts"` while the authorization still works. Fetch the list with that authorization's access token. `authorization.revoked` never carries `linked_accounts_url`.

## Realtime events

Translated OnlyFans activity for one connected linked account, delivered by webhook (when the destination has **Send Realtime events**) or by Server-Sent Events.

| Group | Types |
| --- | --- |
| Messaging | `realtime.chat_message.received`, `realtime.chat_message.liked`, `realtime.chat_message.unliked`, `realtime.chat_typing.updated`, `realtime.inbox_counts.updated` |
| Content | `realtime.post.published`, `realtime.stories.updated`, `realtime.post_fundraising.updated` |
| System | `realtime.system_message.received`, `realtime.system_messages.updated`, `realtime.toasts.received` |

`data` schemas per type: https://docs.ofauth.com/v3/api-reference/realtime/onlyfans-events (each event has its own page). Counter events such as `realtime.inbox_counts.updated` carry current values, not increments. A delayed event does not prove the account is still connected; read linked accounts before acting when status or permissions matter. Before an automatic reply, check that the event was not already handled.

Sandbox simulates activity while a stream is connected, or for 30 minutes after an API request when a verified Sandbox destination receives Realtime events. `POST /v3/sandbox/events` with `{ "type": "<realtime type>" }` (access token plus `x-linked-account-id`) sends one on demand. See [environments.md](environments.md#sandbox).

## Realtime over Server-Sent Events

Server side:

```http
GET https://api.ofauth.com/v3/realtime/events?linkedAccountId=<linked account id>
Accept: text/event-stream
Authorization: Bearer <access_token>
Last-Event-ID: <last id, when resuming>
```

- The stream starts with `retry: 3000` and an `id:` line with no event (keep it as your last event id). Each event has `id` (its `order`), `event` (its type), and `data` (event JSON). `: heartbeat` arrives every 15 seconds.
- Resume with `Last-Event-ID` (or the `lastEventId` query parameter). OFAuth replays the last 15 minutes, up to 1,000 events; otherwise it sends `event: resync`: refetch state, keep listening. Delivery is at least once; skip handled `eventId`s.
- `event: closed` with `data: {"reason": ...}`: `unavailable` (reconnect with `Last-Event-ID` after a short delay; same if the stream ends without `closed`), `reauth` (refresh the token and reconnect once; stop on `401` or `403`), `replaced` (do not reconnect).
- Up to 10 streams per linked account and authorization. When all 10 are server streams, a new one gets `429 stream_limit`.
- Errors before the stream starts: `400` malformed, `401` authentication missing or denied, `406` bad `Accept`, `429 stream_limit`, `503 realtime_unavailable` (retry later).
- Stream envelope: `eventId`, `type`, `schemaVersion`, `order`, `receivedAt`, `environment`, `integrationId`, `linkedAccountId`, `test`, `data`. It differs from the webhook envelope (`receivedAt` instead of `occurredAt`). Lifecycle events never arrive on the stream.

Browser: never send an access token to a page. The backend calls `POST https://api.ofauth.com/v3/realtime` with the bearer token and `{ "linkedAccountId": "..." }`, gets `{ url, expiresAt }` (valid at most 10 minutes), checks the signed-in user may see that account, and returns `url` to the page, which opens `new EventSource(url)` or uses `connectRealtimeStream` from `@ofauth/onlyfans-sdk/realtime-browser`. More than 30 stream URLs per minute per authorization returns `429`. Do not log stream URLs. Guide: https://docs.ofauth.com/v3/guides/realtime-browser.

## Receiver rules

1. Verify the signature on the raw body. Reject failures.
2. Answer `ofauth.webhook.challenge` with the exact challenge as `text/plain`.
3. In one transaction, save the event (deduplicating on `eventId`) and its processing job, then return `2xx` promptly; a worker processes jobs afterwards. Failed deliveries are retried with a wait between attempts.
4. Apply linked-account events in `order` per `linkedAccountId`; retries can arrive out of order. Ignore an event older than the last applied `order` for that account, or read current state.
5. Treat lifecycle events as signals; read `GET /v3/linked-accounts` when you need names, status, or scopes.
6. Check `environment` matches the receiver's configured environment. Keep sample test data out of customer records.
7. Log status, `eventId`, and `type` only; never the secret or full body.
