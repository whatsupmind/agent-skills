# WhatSupMind API reference for agents

Checked against `whatsupmind-gh` revision `d9a6d5f`. Paths below are relative to
the API base URL. Examples assume `API_URL`, `USER_JWT`, `WUM_API_KEY`, and
`SESSION_ID` have been supplied for the intended deployment/session.

## Authentication

| Credential | Header | Scope |
| --- | --- | --- |
| User JWT | `Authorization: Bearer <user-jwt>` | `/api/user/*`: owned sessions, keys, channels, webhooks, HelloMinds settings |
| Session API key | `Authorization: Bearer wum_…` or `X-Api-Key: wum_…` | `/v1/*`: the session identified by the key |
| Admin key | `X-Api-Key: <admin-key>` or bearer header | Administrative endpoints, including `/admin/workers` and `/metrics` |
| Builder key | Managed through HelloMinds settings | WhatSupMind's upstream HelloMinds requests use `X-Access-Key` |

Login/register use `/api/auth/login` and `/api/auth/register` with JSON
`{ "email": "…", "password": "…" }`. The dashboard manages its own login
session. Never substitute a `wum_` key for the user JWT on management routes.

## User management routes

All routes in this section require the user JWT and enforce session ownership.
URL-encode path values such as `chatJid`.

| Method and path | Request / result |
| --- | --- |
| `POST /api/user/sessions` | `{ "label": "Support", "pairingPhone": "+15551234567" }`; omit `pairingPhone` for QR. Returns `201` with `sessionId`, `apiKey`, `keyPrefix`, `qrPending`, `pairingCodeSent`. |
| `GET /api/user/sessions` | List owned sessions. |
| `GET /api/user/sessions/:id` | Session details. |
| `GET /api/user/sessions/:id/status` | Live status and `qrCode` when available. |
| `PATCH /api/user/sessions/:id` | Rename with `{ "label": "New name" }`. |
| `DELETE /api/user/sessions/:id` | Disconnect and delete the session. |
| `POST /api/user/sessions/:id/keys` | Generate another session key; optional `label`. Save returned `apiKey`. |
| `GET /api/user/sessions/:id/keys` | List key metadata, not raw keys. |
| `GET /api/user/sessions/:id/keys/:keyId/reveal` | Reveal an encrypted stored key; older keys may return `409`. |
| `PATCH /api/user/sessions/:id/keys/:keyId` | Update `label`. |
| `DELETE /api/user/sessions/:id/keys/:keyId` | Revoke the key. |
| `GET /api/user/sessions/:id/channels` | All discovered channels, including disabled ones and their Mind settings. |
| `PATCH /api/user/sessions/:id/channels/:chatJid` | Optional `enabled`, `mindEnabled`, `mindId` (string or `null`), `mentionFilter`. |
| `GET /api/user/sessions/:id/feed` | Array from enabled channels, newest first. `limit=1..100` (default 50), `role=user\|assistant\|all`, `before` ISO timestamp with timezone. |
| `POST /api/user/sessions/:id/send` | `{ "chatJid": "…", "text": "…" }` for dashboard sends. |

Channel reads expose `isEnabled`, `mentionFilterEnabled`, `mindEnabled`, and
`mindId`, alongside `chatJid`, `displayName`, `isGroup`, `msgCount`,
`lastMessageAt`, and `lastMessageText`.

### Enable a channel's Mind, then mentions-only

First discover the channel and use its exact JID. This example uses an encoded
group JID; replace it with the intended group's identifier.

```bash
curl -X PATCH "$API_URL/api/user/sessions/$SESSION_ID/channels/120363012345678901%40g.us" \
  -H "Authorization: Bearer $USER_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"enabled":true,"mindEnabled":true,"mindId":null}'

# Only after the previous request succeeds:
curl -X PATCH "$API_URL/api/user/sessions/$SESSION_ID/channels/120363012345678901%40g.us" \
  -H "Authorization: Bearer $USER_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"mentionFilter":true}'

curl "$API_URL/api/user/sessions/$SESSION_ID/channels" \
  -H "Authorization: Bearer $USER_JWT"
```

`mindId: null` means use the account fallback. A channel-specific ID overrides
it. Enable Mind before the mention filter because the filter setter requires
Mind to already be on. Read back persisted values; PATCH echoes supplied fields
and may not reflect a filter coerced off by the setter.

## HelloMinds settings

| Method and path | Contract |
| --- | --- |
| `GET /api/user/hellominds/settings` | `accessKeySet`, `selectedMindId`, `selectedMindName`, `enabled`, and `humanId` when stored. |
| `PUT /api/user/hellominds/settings` | Partial update: `accessKey`, `selectedMindId`, `selectedMindName`, `enabled`. |
| `GET /api/user/hellominds/minds` | `{ "minds": [{ "id": "…", "name": "…", "description": "…", "isEnabled": true }] }`. |

Save the Builder key via `PUT` with `{ "accessKey": "<builder-key>" }`, then
list Minds and choose an enabled one. Example fallback selection:

```bash
curl -X PUT "$API_URL/api/user/hellominds/settings" \
  -H "Authorization: Bearer $USER_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"selectedMindId":"<mind-id>","selectedMindName":"Support","enabled":true}'
```

Empty or null selected-Mind fields clear the fallback. `accessKey: null`
disconnects. Invalid keys return `400`; listing Minds without a key returns
`400`, and upstream list failures return `502`. Use these actual responses to
diagnose an empty selector rather than assuming no Minds exist.

## External V1 API

Every V1 route requires a session `wum_` key. No session ID is needed in the URL.

### Status and enabled channels

```bash
curl "$API_URL/v1/session" -H "Authorization: Bearer $WUM_API_KEY"
curl "$API_URL/v1/channels" -H "Authorization: Bearer $WUM_API_KEY"
```

Status returns `{ sessionId, status, phone, pushName }`. Channels returns
`{ sessionId, channels }`; each channel has `chatJid`, `displayName`, `isGroup`,
`msgCount`, and `lastMessageAt`. Use the user API for Mind/filter settings;
the V1 channel response does not include them.

### Read messages

```bash
curl --get "$API_URL/v1/messages" \
  -H "Authorization: Bearer $WUM_API_KEY" \
  --data-urlencode 'limit=50' \
  --data-urlencode 'channel=120363012345678901@g.us'
```

- `limit`: 1–100, default 50. Values outside that range return `400`.
- `channel`: optional exact chat JID. A disabled/unknown channel returns `400`.
- `after`: the opaque `nextCursor` returned by the previous request. Pass it
  verbatim with URL encoding, never a standalone timestamp.

Response shape:

```json
{
  "sessionId": "support-session",
  "messages": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "chatJid": "120363012345678901@g.us",
      "displayName": "Support Group",
      "isGroup": true,
      "fromJid": "15551234567@s.whatsapp.net",
      "content": "Can you help?",
      "waMessageId": "ABCD1234",
      "createdAt": "2026-10-13T07:00:00.000Z"
    }
  ],
  "nextCursor": "2026-10-13T07:00:00.000Z|550e8400-e29b-41d4-a716-446655440000",
  "hasMore": false
}
```

Request more pages while `hasMore` is true. For ongoing polling, retain the last
non-null cursor. With no enabled channels, the response is an empty `messages`
array, `nextCursor: null`, and `hasMore: false`.

The dashboard feed uses a separate `before` timestamp cursor and includes
`role`; do not interchange the two pagination schemes.

### Send or reply

```bash
curl -X POST "$API_URL/v1/messages/send" \
  -H "Authorization: Bearer $WUM_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"chatJid":"120363012345678901@g.us","text":"Got it, thanks!"}'
```

The public request accepts `chatJid` and `text`. It does not implement
`mediaUrl`, base64 uploads, or an `attachments` request field. Use exact
discovered JIDs; do not convert LIDs into phone numbers.

| Status | Meaning |
| --- | --- |
| `200` | `{ "ok": true, "queued": false, "sessionId": "…", "chatJid": "…" }` — send confirmed by the worker. |
| `202` | Same shape with `queued: true` — queued for worker recovery, not delivery confirmation. |
| `400` | Required `chatJid` or `text` missing. |
| `401` | Missing, invalid, or revoked session credentials. |
| `429` | Per-session rate limit; `Retry-After: 60`. Default configured limit is 30/minute. |
| `502` | Worker explicitly rejected the send. |
| `500` | Other send error. |

Queued messages default to a five-minute validity window
(`OUTBOX_MESSAGE_TTL_MS=300000`). There is no idempotency field in this request;
do not automatically repeat a queued or ambiguously completed send.

## Webhook management and payloads

These are user-JWT routes, not V1 routes:

| Method and path | Purpose |
| --- | --- |
| `POST /api/user/sessions/:id/webhooks` | Register `{ "url": "https://receiver.example.com/hook", "label": "Automation" }`. Returns `201` with metadata and `secret`. |
| `GET /api/user/sessions/:id/webhooks` | List webhook metadata, excluding secrets. |
| `GET /api/user/sessions/:id/webhooks/:wid/secret` | Reveal `{ "secret": "…" }`. |
| `PATCH /api/user/sessions/:id/webhooks/:wid` | Update `url`, `label`, or `enabled`. |
| `DELETE /api/user/sessions/:id/webhooks/:wid` | Delete webhook and stop its pending retries. |
| `POST /api/user/sessions/:id/webhooks/:wid/ping` | Explicit signed test; returns `{ ok, status }` for an HTTP response, or `502` on a fetch failure. |

Example message POST body:

```json
{
  "sessionId": "support-session",
  "chatJid": "120363012345678901@g.us",
  "displayName": "Alice",
  "isGroup": true,
  "from": "15551234567@s.whatsapp.net",
  "text": "Can you help?",
  "messageId": "ABCD1234",
  "timestamp": "2026-10-13T07:00:00.000Z",
  "mentionedJids": [],
  "quotedText": "",
  "subject": "💬 Alice"
}
```

The handler derives `displayName` from the sender's push name or sender ID; do
not rely on it as a group identity. Use `chatJid`. `messageId` can be empty if
unavailable. The payload does not carry binary attachments or a reply token.

An explicit ping uses this separate shape:

```json
{
  "type": "ping",
  "webhookId": "550e8400-e29b-41d4-a716-446655440000",
  "sessionId": "support-session",
  "timestamp": "2026-10-13T07:00:00.000Z"
}
```

### Verify before parsing

Node.js (supply the unparsed request bytes and the signature header):

```js
import { createHmac, timingSafeEqual } from 'node:crypto';

function verifyWumSignature(rawBody, signatureHeader, secret) {
  if (typeof signatureHeader !== 'string') return false;
  const expected = Buffer.from(
    'sha256=' + createHmac('sha256', secret).update(rawBody).digest('hex'),
    'utf8'
  );
  const received = Buffer.from(signatureHeader, 'utf8');
  return received.length === expected.length && timingSafeEqual(received, expected);
}
```

Python:

```python
import hashlib
import hmac

def verify_wum_signature(raw_body: bytes, header: str, secret: str) -> bool:
    if not isinstance(header, str):
        return False
    expected = b"sha256=" + hmac.new(
        secret.encode("utf-8"), raw_body, hashlib.sha256
    ).hexdigest().encode("ascii")
    return hmac.compare_digest(expected, header.encode("utf-8"))
```

In Express, use `express.raw({ type: 'application/json' })` on the receiver
before any JSON body parser. Parse only after verification. Missing, malformed,
or wrong-length headers must fail verification rather than throw.

Return `2xx` within ten seconds. Message delivery failures retry on the logged
schedule of 30, 60, 300, then 1800 seconds; failure five becomes `dead_letter`.
Pings are explicit connectivity checks, not this message-delivery retry flow.
