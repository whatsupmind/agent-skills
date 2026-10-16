---
name: whatsupmind-operator-manual
description: Use when operating WhatSupMind, signing in with HelloMinds OAuth, linking WhatsApp sessions, configuring Mind auto-replies, integrating signed webhooks or the REST API, or troubleshooting authorization and message delivery.
metadata:
  source-branch: main
  source-revision: ddf6113c13756e01a20cdb7839633268408097ec
---

# WhatSupMind — Operator Manual

WhatSupMind links WhatsApp devices, forwards selected chats to signed webhooks,
and connects them to AI Minds built at [HelloMinds](https://hellominds.ai).
Use the dashboard for setup and the REST API for external integrations.

This manual is checked against `whatsupmind-gh` **main**, revision `ddf6113`.
For exact requests, responses, and webhook verification, read
[references/api.md](references/api.md).

## Choose the workflow

- **AI auto-replies:** launch and enable a Mind at HelloMinds, sign in through
  HelloMinds OAuth, link WhatsApp, and configure the account and channel switches below.
- **Webhooks or custom automation:** link WhatsApp, enable the channels, register
  a webhook, and use the session's V1 API key to send replies. A Mind is optional.
- **Troubleshooting:** identify the session, chat JID, and failing stage: linking,
  channel visibility, inbound forwarding, Mind response, or WhatsApp send.

Use the deployment's actual API and dashboard URLs. Examples use
`https://api.whatsupmind.com` and `https://dashboard.whatsupmind.com`;
development defaults are `http://localhost:3000` and `http://localhost:5173`.
Keep the WhatSupMind user JWT, session `wum_` key, HelloMinds OAuth tokens, and
webhook signing secret distinct; each authenticates a different operation.
HelloMinds uses OAuth only: do not ask users to generate or paste Builder API keys.

## Sessions and channels

- **Session:** one linked WhatsApp device, with one or more session-scoped API keys.
- **Channel:** one DM or group within a session, identified by its `chatJid`.
- **HelloMinds settings:** account-wide settings associated with the signed-in
  user's email, shared across that user's sessions.

Channels start disabled and Mind auto-reply starts off. Inbound messages are
stored and channels discovered even while disabled, but disabled channels do
not forward new messages to webhooks, Redis, or HelloMinds. The Live Feed and
V1 message reads expose only currently enabled channels. Enabling a channel can
therefore make previously stored messages visible; it does not replay them to
webhooks or the Mind.

Groups are also discovered when WhatsApp connects and when the account joins a
group. Leaving or being removed from a group removes its channel. If it later
reappears, check and reconfigure its settings.

### Recent history and session removal

Every WhatsApp connection requests the recent-message batch, not the full chat
catalog. Imported history is deduplicated and preserves WhatsApp send times;
the seven-day import window is an upper bound, not a promise of seven full days
of history. Historical messages do not trigger Mind replies or webhooks.
Discovered channels remain disabled until enabled. Raw-JID/phone-number names
can improve as synced contacts and targeted name-healing history arrive.

Explicit session deletion hard-deletes its row and child data (messages, keys,
channels, webhooks, and delivery logs), with best-effort WhatsApp unpairing.
Phone logout or exhausted reconnect attempts instead marks the session
`unlinked` and preserves its data. There is no relink action for that session;
create a new session to link WhatsApp again.

## Link WhatsApp

1. Sign in to the dashboard and open the new-session/link-device flow.
2. Enter a session name and choose **QR Code** or **Phone Number**.
3. For QR, open WhatsApp → **Linked Devices** → **Link a Device** on the phone
   and scan the dashboard QR.
4. For phone pairing, provide the WhatsApp phone number with country code, then
   enter the generated pairing code in WhatsApp's phone-number linking flow.
   The current create response exposes `pairingCodeSent`, not the code itself;
   if the deployment does not display the code, use QR linking.
5. Wait for `connected` and save the returned `wum_` API key.

The linking UI polls every two seconds and stops after about two minutes.
Retry the linking flow if it expires. After linking, the API hands the connection
to a session worker. If it immediately drops, check worker and Redis availability
as well as the phone's linked-device state.

Keys can be generated, revealed when encrypted storage is available, and revoked
in the session's API key controls. To rotate, create a replacement, update the
integration, then revoke the old key.

## Sign in with HelloMinds and select a Mind

1. Build, publish, and enable the Mind at [HelloMinds](https://hellominds.ai).
2. Use the dashboard's **Sign in with HelloMinds** flow and approve email,
   Mind, conversation, and messaging permissions. Login and forwarding use
   one OAuth consent; there is no separate key-pasting or forwarding-only connect step.
3. The browser completes PKCE at `/hellominds/callback`, exchanges the HelloMinds
   identity for a WhatSupMind login, and stores the forwarding token bundle on
   the API before opening the dashboard.
4. Open the session detail page's **HelloMinds** panel. Check the selected
   fallback Mind and **Fallback enabled** state. Disabled Minds are hidden.
5. Enable the desired channel, then select its Mind. Selecting a specific Mind
   or **Use fallback** also enables the channel's Mind switch. Clearing the
   selection disables it; the switch cannot enable an unselected Mind in the UI.
6. Test with an inbound WhatsApp message from another account, then check the
   Live Feed and receiving WhatsApp chat.

The API stores encrypted access/refresh tokens and refreshes them server-side
near expiry. The browser can be closed while forwarding continues. If the
grant is missing, revoked, or expired, **sign in again through HelloMinds** and
renew consent. Do not manually reuse rotating refresh tokens.

### All forwarding gates must pass

| Setting | Meaning |
| --- | --- |
| Channel `isEnabled` | Allows downstream processing of new inbound messages. Write it as `enabled` in the API. |
| Channel `mindEnabled` | Allows that channel's messages to be sent to HelloMinds. |
| Account `oauthConnected` | An OAuth refresh token is stored. Check `scope` and actual API responses for authorization health; tokens are not returned by settings reads. |
| Account `enabled` | Labeled **Fallback enabled** in the UI; the current handler still gates all new HelloMinds forwards on it, including channel-specific Minds. |
| Effective Mind | Channel `mindId` takes precedence; `null` uses account `selectedMindId`. |

The OAuth token-storage handler normally enables account forwarding and attempts
to select the first enabled Mind if none is set. A migrated row's explicitly
disabled state can be carried over. Read settings after login rather than
assuming the fallback is off or a Mind was successfully selected. Later changes
to the fallback selection do not themselves toggle account `enabled`.

The channel switch changes only `isEnabled`; Mind enablement is separate in
both UI and REST. **Use fallback** in the UI copies the fallback's concrete ID
onto the channel, so later fallback changes do not retarget it automatically.
The REST API still permits `mindId: null` to dynamically resolve the account
fallback, but the UI requires an explicit selection before enabling Mind.
Always read back channel settings after changes.

### Group mention filtering

**Mentions only** is optional and defaults to **off**. Enable it for groups
where the linked WhatsApp account should only process explicit @mentions.
It is ignored for DMs. The comparison supports both phone JIDs and WhatsApp LIDs,
including device-suffixed identifiers.

The filter runs before **webhook delivery, Redis publication, and HelloMinds
forwarding**. It does not filter storage or message reads. When active, numeric
@mentions are stripped from forwarded `text`; `mentionedJids` and `quotedText`
remain in the webhook payload. If both linked-account identities are unknown,
the current handler logs a warning and processes the message in degraded mode.

Enable Mind first, then enable the mention filter in a separate API request:
the handler applies `mentionFilter` before `mindEnabled` within one PATCH.
Turning Mind off clears the filter; turning Mind back on does not restore it.
The read field is `mentionFilterEnabled`; the write field is `mentionFilter`.

### How Mind replies arrive

WhatSupMind sends messages to HelloMinds with sender context and receives Mind
replies through **alias-scoped server-sent events (SSE)**. Workers subscribe to
`/v1/messaging/events?alias=<conversation-alias>` with a fresh OAuth bearer token,
using the SDK's canonical API base URL. There is one stream/ownership lease per
account and alias. The bridge waits for successful stream headers before sending
the message upstream. An unfiltered stream returns `403` for external OAuth
callers even with `messaging:stream` permission. Priming messages, reply tokens,
HTTP reply callbacks, and history polling are no longer the auto-reply workflow.

Conversation aliases are stable hashes of the user's email, chat JID, and Mind
ID: `wum-{emailHash}-{jidHash}-{mindHash}`. They do not rotate daily. Changing
the effective Mind selects that Mind's alias, creating its conversation if
needed; switching back can reuse an existing conversation.

Incoming replies route by the user/chat prefix, ignoring the Mind hash and
legacy date suffix. Thus a late reply from the previous Mind can still reach
the channel after a switch. The same chat enabled in multiple sessions for the
same user can receive the reply in each matching session. Routing targets require
both channel enablement and Mind enablement. The account master switch gates
new forwards; it is not a cancellation mechanism for replies already in flight.

The stream rejects events for a different alias and uses the subscription alias
when an event omits it. WhatsApp dispatch waits for the Live Feed insert;
duplicate message IDs (or fingerprints when IDs are absent) suppress repeat
sends across streams/workers. A failure after insertion can still leave a feed
entry without WhatsApp delivery, so the feed alone is not a delivery receipt.

Inbound WhatsApp attachments are forwarded to HelloMinds as base64 when the
downloaded buffer, MIME type, and filename are available. SSE replies support
text, attachments, or attachments alone; HTML is converted to WhatsApp text.
SSE attachments over 10 MiB are skipped. Mind replies are recorded as assistant
messages in the Live Feed; feed presence alone does not confirm WhatsApp delivery.
This internal media support does not add media fields to the public V1 send API.

## Webhooks and custom replies

Open the session's webhooks controls, register an endpoint, and save its signing
secret. The user API can reveal the stored secret again. There is no dedicated
secret-rotation endpoint; replace the webhook when a new secret is needed.
Use **Ping** or the explicit ping API to test it.

Verify `X-WUM-Signature: sha256=<hex>` against the raw request body with
HMAC-SHA256 and a length-checked, constant-time comparison. See the working
examples and payload in [references/api.md](references/api.md).

Return `2xx` within ten seconds and process work asynchronously. Logged message
deliveries retry after failures with delays of **30, 60, 300, and 1800 seconds**;
the fifth failed attempt becomes `dead_letter`. These are scheduling delays,
so worker availability also affects timing. Handle repeat deliveries using
`sessionId` and a nonempty `messageId` for deduplication.

Message payloads contain `subject` but no `replyToken` and no `event` wrapper.
Explicit test pings use `type: "ping"`. To reply from your own code, send
`{ "chatJid": "<payload.chatJid>", "text": "Your reply" }` to
`POST /v1/messages/send` with that session's `wum_` key. In a group, reply to
`chatJid`, not the individual sender's `from` value.

**Migration from older integrations:** `/v1/reply`, the old `/webhook` reply
callback, reply-token subjects, email-notification replies, and Mind priming
are absent from the current runtime. Migrate custom replies to authenticated
V1 sends. If custom automation should be the only responder, turn off the
channel's Mind switch.

## HelloMinds OAuth deployment setup

Register this exact redirect URI for the HelloMinds OAuth client, using the
dashboard's actual browser origin:

```text
https://dashboard.whatsupmind.com/hellominds/callback
```

The browser builds it from `window.location.origin`; localhost and 127.0.0.1
are different origins. Configure:

- Web build: `VITE_MINDS_CLIENT_ID` — public OAuth client UUID.
- API/workers: `MINDS_CLIENT_ID` — the same UUID for refresh, revoke, and audience checks.
- API/workers: `MINDS_AUTH_BASE_URL` — defaults to `https://api.oauth.hellominds.ai`;
  token/revoke endpoints are `/v2/oauth/token` and `/v2/oauth/revoke`.
- API/workers: `ENCRYPTION_KEY` — encrypts the stored forwarding token bundle.
- API: `MINDS_JWT_PUBLIC_KEY` — HelloMinds signing public key for signature verification;
  without it the implementation falls back to structural token checks.
- API: `FRONTEND_URL` — dashboard origin used by the CORS allowlist.

The browser requests `Email`, `MindsList`, `MindsStatus`, `ConversationsCreate`,
`ConversationsList`, `ConversationsRead`, `MessagingSend`, `MessagingHistory`,
and `MessagingStream` scopes from Minds Connect. Use the full set; default
OAuth scopes alone do not include all forwarding permissions.

HelloMinds is the only current login mechanism. Email/password registration,
direct Google/GitHub/X login routes, and their provider environment variables
are removed. `HELLOMINDS_BASE_URL` is also no longer read; messaging uses the
SDK's canonical `API_BUILD_BASE_URL`.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Channel missing from V1 or feed | Use the user API channel list, which includes disabled channels. Enable it. An explicit V1 filter for a disabled/unknown channel returns `400`. |
| New group absent or old group vanished | Confirm actual WhatsApp membership and worker connection. Groups are registered on connection/join and removed on exit. |
| Mind never replies | Check channel enabled → group mention gate → channel Mind enabled → `oauthConnected` and scopes → account `enabled` (Fallback toggle) → selected Mind enabled upstream. |
| Mind selector empty | Inspect `GET /api/user/hellominds/minds`. `401` with `HELLOMINDS_LOGIN_REQUIRED` means sign in again; transient `502` is not proof of revoked consent. Disabled Minds are hidden. |
| Toggles appear to revert | Read back user API settings and inspect the failed PATCH. The dashboard uses optimistic state reconciled by polling; visual state alone is not confirmation. |
| Forward logged but no Mind response | Check the worker's SSE connection, upstream response, and `alias_prefix` routing. Look for `no active targets for channel` or `failed to persist alias_prefix`; another inbound message retries prefix persistence. |
| Replies stop after worker restart | Check account/alias SSE leases, Redis, fresh OAuth tokens, and stream-open logs. Recovery reopens alias leases; stale ownership can delay a new lazy open until expiry. |
| SSE returns `403` | Confirm the subscription includes the conversation `alias` and the OAuth grant includes messaging-stream permission; a user-wide subscription is rejected. |
| Login worked but forwarding later fails | Check `MINDS_CLIENT_ID`, token decryption, and refresh errors. A terminal `invalid_grant` clears the bundle and invalidates the app session; sign in again. Network/5xx/config errors preserve the bundle for retry. |
| Reply reaches multiple sessions or comes from the previous Mind | Check user/chat alias routing and the enabled channels in each session; routing ignores Mind hash. |
| Attachment missing | Check inbound media download and metadata, or the SSE attachment format and 10 MiB cap. Public V1 sends accept text only. |
| Webhook quiet | Check channel enablement, mention filter, webhook enabled state, ping result, and delivery logs. `dead_letter` means automatic retries are exhausted. |
| Signature fails | Verify raw bytes before JSON parsing, the full `sha256=` header, and the correct webhook secret. |
| Send returns `202` | It is queued, not delivered. Check session-worker recovery; the default outbox message lifetime is five minutes. Avoid blindly resending. |
| Send returns `429` | Honor `Retry-After: 60`. Default session send limit is 30/minute, configured server-side by `WA_RATE_LIMIT_PER_MIN`. |
| OAuth redirect mismatch | Compare the HelloMinds client registration with the actual browser origin plus `/hellominds/callback`; confirm matching web/API client IDs. |

For deployment diagnostics, `/health` checks API liveness; `/ready` checks
Postgres and Redis and returns `503` on dependency failure. A healthy API alone
does not establish that a session worker or HelloMinds SSE stream is connected.
Admins can inspect `/admin/workers` and authenticated `/metrics`.

Useful current server defaults: `MINDS_AUTH_BASE_URL=https://api.oauth.hellominds.ai`,
`SSE_OWNER_TTL_SECONDS=300`, `SSE_ROUTING_CACHE_TTL_SECONDS=120`,
`OWNER_TTL_SECONDS=120`, `WORKER_HEARTBEAT_INTERVAL_MS=60000`, and
`WORKER_CLAIMER_INTERVAL_MS=40000` plus jitter. Do not promise 30-second failover.

## Keeping this skill current

Check the source branch before updating: the local checkout may be the legacy
`old` branch. Use current `origin/main` and record its exact revision unless the
user explicitly targets a legacy deployment. Use the deployed version's handlers
as the authority when documentation conflicts.
The dashboard `/docs`, `docs/api.md`, and `docs/webhook-payload.md` may retain
legacy examples. In `whatsupmind-gh`, compare:

- `apps/api/src/api/{server,auth,user-sessions,v1}.ts` — routes and contracts.
- `apps/api/src/lib/{message-handler,hellominds-api,sse-bridge,sse-routing-cache}.ts`
  — forwarding, media, and response routing.
- `apps/api/src/db/repositories/{channels,webhook-delivery-log}.ts` — settings
  interactions and retry schedule; migrations establish initial defaults.
- `apps/api/src/lib/{hellominds-token-provider,minds-jwt}.ts`,
  `apps/web/src/lib/minds-oauth.ts`, `apps/web/src/pages/HelloMindsCallback.tsx`,
  and `apps/api/src/config.ts` — OAuth lifecycle, callback, and configuration.
- `docs/adr/` and `apps/api/src/lib/history-sync.ts` — history, deletion, and SSE decisions.
- `apps/web/src/components/{CreateSessionModal,HelloMindsSettings,ChannelToggle}.tsx`
  — actual dashboard labels and behavior.

Update this skill and its API reference together, then advance `source-revision`.
