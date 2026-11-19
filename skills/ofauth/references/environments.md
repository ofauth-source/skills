# Environments

Full reference: https://docs.ofauth.com/v3/api-reference/environments. Setup guides: https://docs.ofauth.com/v3/setup/create-integration, https://docs.ofauth.com/v3/setup/sandbox, https://docs.ofauth.com/v3/setup/integration-checklist.

Every Developer integration (External Integration) has three fully isolated environments. Each has its own client ID and secret, redirect URIs, linked accounts, tokens, webhook destinations, and data. The client ID and secret select the environment; the API hostname is the same, and there is no environment or `resource` parameter. Operator private apps register their own client instead; see [oauth.md](oauth.md#operator-private-apps).

| | Sandbox | Test | Live |
| --- | --- | --- | --- |
| Client ID | `ofext_sandbox_...` | `ofext_test_...` | `ofext_live_...` |
| Data | Sample data. No OnlyFans calls. Common writes kept for 24 hours. | Real. Only the integration owner's designated test accounts. | Real customer accounts. |
| Who can authorize | Members of the integration's organization who can manage its credentials, for that organization's Sandbox accounts. | Members of the organization, using designated test accounts. | Any OFAuth user. |
| Limits | Standard rate limits. | Standard rate limits plus a daily operation quota. | Standard rate limits. |
| Redirect URIs | HTTPS, or `http://127.0.0.1` / `http://[::1]` loopback. | Same as Sandbox. | HTTPS with a domain hostname only. |
| Consent purpose | Not shown. | Not shown. | Shown from the approved app configuration (never a request parameter). |
| Approval | None. | None. | Live approval of the integration. |

Tokens work only in the environment of the client that issued them.

## Configuration

Keep one configuration object per environment, read from env vars. Adapt names to the codebase.

```typescript
type OFAuthEnvironment = "sandbox" | "test" | "live"

const ofauth: Record<OFAuthEnvironment, {
  clientId?: string
  clientSecret?: string
  redirectUri?: string
  webhookSecret?: string
}> = {
  sandbox: {
    clientId: process.env.OFAUTH_SANDBOX_CLIENT_ID,
    clientSecret: process.env.OFAUTH_SANDBOX_CLIENT_SECRET,
    redirectUri: process.env.OFAUTH_SANDBOX_REDIRECT_URI, // e.g. http://127.0.0.1:3000/oauth/ofauth/callback
    webhookSecret: process.env.OFAUTH_SANDBOX_WEBHOOK_SECRET
  },
  test: {
    clientId: process.env.OFAUTH_TEST_CLIENT_ID,
    clientSecret: process.env.OFAUTH_TEST_CLIENT_SECRET,
    redirectUri: process.env.OFAUTH_TEST_REDIRECT_URI,
    webhookSecret: process.env.OFAUTH_TEST_WEBHOOK_SECRET
  },
  live: {
    clientId: process.env.OFAUTH_LIVE_CLIENT_ID,
    clientSecret: process.env.OFAUTH_LIVE_CLIENT_SECRET,
    redirectUri: process.env.OFAUTH_LIVE_REDIRECT_URI, // https://app.example.com/oauth/ofauth/callback
    webhookSecret: process.env.OFAUTH_LIVE_WEBHOOK_SECRET
  }
}
```

Example env file entries (placeholders only; the user fills real values from the dashboard):

```sh
OFAUTH_SANDBOX_CLIENT_ID=
OFAUTH_SANDBOX_CLIENT_SECRET=
OFAUTH_SANDBOX_REDIRECT_URI=
OFAUTH_SANDBOX_WEBHOOK_SECRET=
```

Rules:

- Never write real client secrets or webhook signing secrets into files. Never change existing values in a `.env` file. Keep `.env` out of version control.
- Validate at startup that the client ID prefix matches the configured environment.
- Store the client ID (environment) next to every token, linked account id, account selection, and webhook secret. A Sandbox token cannot reach a Live account, and a Live id does not exist in Sandbox.
- The callback host should match the app host so the initiating session is available. Configure the framework's trusted origin to match.

## Sandbox

For development, CI, and demos. Authorization works end to end with sample accounts the user creates in the dashboard under the integration's Sandbox environment. Never contacts OnlyFans. Vault+ is always on.

- Each Sandbox authorization shares exactly one account. To share another, run authorization again; it is added to the same authorization. A refused selection returns `error=access_denied` with `error_description`.
- Sample data is stable between requests (dates move with the current time) and has the same fields as Live.
- Writes to chats, Vault lists, mass messages, and stories are kept for 24 hours per linked account (capped), so read-after-write works. Do not use Sandbox as storage. Uploads up to 1 MiB run the full flow; bytes are discarded.
- Reconnect testing: **Expire** an account in the dashboard (status `needs_reconnect`, sends `linked_account.expired`), then reconnect via `POST /v3/linked-accounts/{id}/reconnect` or the dashboard's **Reconnect** (sends `linked_account.reconnected`). **Delete** sends `linked_account.disconnected`.
- Realtime is simulated; `POST /v3/sandbox/events` sends an event on demand (up to 300 events per account per rolling hour; over that, `429 sandbox_event_limit`). Sandbox never sends `realtime.post.published`. See [webhooks.md](webhooks.md#realtime-events).

Sample data cannot prove the app handles real account states; use Test for that.

## Test

Real OnlyFans accounts the integration owner designates in the dashboard. Sign in as an organization member and approve the designated account. A fixed daily number of Access operations; when used up, `429 quota_exhausted` with `Retry-After` until the UTC reset.

Link each test account once and rely on refresh; do not put real-account authorization in automated tests. Use Sandbox in CI.

## Live

Requirements:

1. An active Developer plan on the organization. Without it: `subscription_required`.
2. Completed public app details and Live approval of the integration, requested from the dashboard once it works in Test. Review covers the purpose statement, scopes, redirect URIs, and privacy policy. Until approved: `integration_not_approved`.
3. HTTPS redirect URIs with domain hostnames, added in the integration's Live settings.
4. The approved purpose, confirmed in the dashboard; OFAuth shows it during Live consent.

The user requests approval and manages the plan; you cannot do either in code.

## Go-live checklist

- [ ] Sandbox: full authorize, callback (state, session, `iss`), exchange, account selection, Access call, refresh, disconnect.
- [ ] Declined consent lets the user start again.
- [ ] Refresh persists the new access token; stale-write protection on reconnect or disconnect; `invalid_grant` leads to re-authorization.
- [ ] Webhook receiver verifies signatures, answers the challenge, deduplicates on `eventId`, handles all six lifecycle events, and separates environments.
- [ ] Errors branch on `recovery.action`, including re-authorization, reconnect, and additional consent. Unknown write outcomes are checked, not retried blindly.
- [ ] Test: one designated real account linked once; real Access calls succeed within the quota.
- [ ] Live: client ID and secret stored as secrets, HTTPS callback registered, scopes minimal, Developer plan active, Live approval granted, Live webhook destination verified and active. Secure cookies; browser sessions in a store shared by app instances.
- [ ] No secrets, tokens, verifiers, or stream URLs in the repo, logs, or browser.
