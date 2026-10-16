---
name: whatsupmind-operator-manual
description: Use when operating WhatSupMind, linking WhatsApp sessions, configuring HelloMinds Mind auto-replies, integrating signed webhooks or the REST API, or troubleshooting message delivery and OAuth login.
metadata:
  source-revision: d9a6d5f46af65d4140a1b74b7cb83397b5bd50e4
---

# WhatSupMind — Operator Manual

WhatSupMind links WhatsApp devices, forwards selected chats to signed webhooks,
and connects them to AI Minds built at [HelloMinds](https://hellominds.ai).
Use the dashboard for setup and the REST API for external integrations.

This manual is checked against `whatsupmind-gh` revision `d9a6d5f`.
For exact requests, responses, and webhook verification, read
[references/api.md](references/api.md).

## Choose the workflow

- **AI auto-replies:** launch and enable a Mind at HelloMinds, obtain a Builder
  API key, link WhatsApp, and configure the account and channel switches below.
- **Webhooks or custom automation:** link WhatsApp, enable the channels, register
  a webhook, and use the session's V1 API key to send replies. A Mind is optional.
- **Troubleshooting:** identify the session, chat JID, and failing stage: linking,
  channel visibility, inbound forwarding, Mind response, or WhatsApp send.

Use the deployment's actual API and dashboard URLs. Examples use
`https://api.whatsupmind.com` and `https://dashboard.whatsupmind.com`;
development defaults are `http://localhost:3000` and `http://localhost:5173`.
Keep the user JWT, session `wum_` key, HelloMinds Builder key, and webhook signing
secret distinct; each authenticates a different operation.

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

## Connect a HelloMinds Mind

1. Build, publish, and enable the Mind at [HelloMinds](https://hellominds.ai).
2. Generate a **Builder API key** at
   [the Builder console](https://build.hellominds.ai/console?setup=1).
3. Open the session detail page's **HelloMinds** settings, choose **Connect**,
   paste the key, and select **Save Key**.
4. Confirm **Forwarding enabled** and choose an enabled **fallback Mind** if
   channels should inherit a default. Disabled Minds are hidden from selectors.
5. Enable the desired channel and its separate **Mind** switch. Select a
   channel-specific Mind, or choose **use fallback**.
6. Test with an inbound WhatsApp message from another account, then check the
   Live Feed and the receiving WhatsApp chat.

### All forwarding gates must pass

| Setting | Meaning |
| --- | --- |
| Channel `isEnabled` | Allows downstream processing of new inbound messages. Write it as `enabled` in the API. |
| Channel `mindEnabled` | Allows that channel's messages to be sent to HelloMinds. |
| Account `accessKeySet` | A Builder key has been saved. The settings response does not expose the key. |
| Account `enabled` | Master switch for forwarding new messages to HelloMinds. |
| Effective Mind | Channel `mindId` takes precedence; `null` uses account `selectedMindId`. |

Adding or rotating a Builder key defaults account forwarding to enabled unless
`enabled` is explicitly supplied. Resubmitting the same key preserves the
existing switch. Disconnecting with `accessKey: null` clears the connection.

The dashboard's channel switch also changes `mindEnabled` when a Builder key is
configured. The REST channel `enabled` field alone does not do that. Always
read back channel settings after changes. Selecting a Mind does not itself
enable the channel's Mind switch.

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
replies through a **server-sent events (SSE) bridge**. Workers manage the stream
at the HelloMinds API's `/v1/messaging/events` using the Builder key. A browser
does not need to stay open. Priming messages, reply tokens, HTTP reply callbacks,
and history polling are no longer the auto-reply workflow.

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

## OAuth setup (Google, GitHub, X)

Register this exact provider redirect URI on the **dashboard** domain:

```text
https://dashboard.whatsupmind.com/auth/callback/<provider>
```

`<provider>` is `google`, `github`, or `twitter`. The code derives the URI from
`FRONTEND_URL`, so changing the dashboard domain also requires updating the
provider's registered callback. The frontend exchanges the callback code via
`POST /api/auth/oauth/:provider/exchange` on the API.

Set these API-server variables for the providers you enable:

- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
- `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`
- `TWITTER_CLIENT_ID`, `TWITTER_CLIENT_SECRET`
- `FRONTEND_URL` — exact dashboard origin, without a trailing slash; also used
  by the API's browser CORS allowlist.

The variables do not have an `OAUTH_` prefix. `API_BASE_URL` is the API's public
base URL and does not determine the provider redirect URI.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Channel missing from V1 or feed | Use the user API channel list, which includes disabled channels. Enable it. An explicit V1 filter for a disabled/unknown channel returns `400`. |
| New group absent or old group vanished | Confirm actual WhatsApp membership and worker connection. Groups are registered on connection/join and removed on exit. |
| Mind never replies | Check channel enabled → group mention gate → channel Mind enabled → saved Builder key → account forwarding enabled → channel Mind or fallback selected and enabled upstream. |
| Mind selector empty | Check Builder key validity and `GET /api/user/hellominds/minds`; disabled Minds are hidden by the dashboard. |
| Toggles appear to revert | Read back user API settings and inspect the failed PATCH. The dashboard uses optimistic state reconciled by polling; visual state alone is not confirmation. |
| Forward logged but no Mind response | Check the worker's SSE connection, upstream response, and `alias_prefix` routing. Look for `no active targets for channel` or `failed to persist alias_prefix`; another inbound message retries prefix persistence. |
| Replies stop after worker restart | Check SSE ownership and Redis. Streams open lazily on forwarded messages; startup recovery only checks bridges owned by the current worker ID. After stale ownership expires, test another inbound message and confirm the stream opens. |
| Reply reaches multiple sessions or comes from the previous Mind | Check user/chat alias routing and the enabled channels in each session; routing ignores Mind hash. |
| Attachment missing | Check inbound media download and metadata, or the SSE attachment format and 10 MiB cap. Public V1 sends accept text only. |
| Webhook quiet | Check channel enablement, mention filter, webhook enabled state, ping result, and delivery logs. `dead_letter` means automatic retries are exhausted. |
| Signature fails | Verify raw bytes before JSON parsing, the full `sha256=` header, and the correct webhook secret. |
| Send returns `202` | It is queued, not delivered. Check session-worker recovery; the default outbox message lifetime is five minutes. Avoid blindly resending. |
| Send returns `429` | Honor `Retry-After: 60`. Default session send limit is 30/minute, configured server-side by `WA_RATE_LIMIT_PER_MIN`. |
| OAuth redirect mismatch | Compare the provider registration with `FRONTEND_URL/auth/callback/<provider>` exactly. |

For deployment diagnostics, `/health` checks API liveness; `/ready` checks
Postgres and Redis and returns `503` on dependency failure. A healthy API alone
does not establish that a session worker or HelloMinds SSE stream is connected.
Admins can inspect `/admin/workers` and authenticated `/metrics`.

Useful current server defaults: `HELLOMINDS_BASE_URL=https://api.build.hellominds.ai`,
`SSE_OWNER_TTL_SECONDS=300`, `SSE_ROUTING_CACHE_TTL_SECONDS=120`,
`OWNER_TTL_SECONDS=120`, `WORKER_HEARTBEAT_INTERVAL_MS=60000`, and
`WORKER_CLAIMER_INTERVAL_MS=40000` plus jitter. Do not promise 30-second failover.

## Keeping this skill current

Use the deployed version's handlers as the authority when documentation conflicts.
The dashboard `/docs`, `docs/api.md`, and `docs/webhook-payload.md` may retain
legacy examples. In `whatsupmind-gh`, compare:

- `apps/api/src/api/{server,user-sessions,v1}.ts` — routes and contracts.
- `apps/api/src/lib/{message-handler,hellominds-api,sse-bridge,sse-routing-cache}.ts`
  — forwarding, media, and response routing.
- `apps/api/src/db/repositories/{channels,webhook-delivery-log}.ts` — settings
  interactions and retry schedule; migrations establish initial defaults.
- `apps/api/src/lib/oauth/callback-url.ts` and `apps/api/src/config.ts` — callback
  construction, environment names, and defaults.
- `apps/web/src/components/{CreateSessionModal,HelloMindsSettings,ChannelToggle}.tsx`
  — actual dashboard labels and behavior.

Update this skill and its API reference together, then advance `source-revision`.
