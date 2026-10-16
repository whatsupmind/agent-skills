# WhatSupMind Agent Skills

[![skills.sh](https://skills.sh/b/whatsupmind/agent-skills)](https://skills.sh/whatsupmind/agent-skills)

Agent skills for [WhatSupMind](https://dashboard.whatsupmind.com) — link WhatsApp devices, configure AI Mind auto-replies, and integrate with signed webhooks and the REST API.

## Installation

```bash
npx skills add whatsupmind/agent-skills
```

## Skills

### whatsupmind-operator-manual

Operator manual for WhatSupMind. Covers:

- Linking WhatsApp sessions via QR code or pairing code
- Signing in through HelloMinds OAuth, server-managed token refresh, and renewed consent
- Configuring account forwarding, per-channel Minds, and fallback selection
- Understanding group mention filtering, alias-scoped SSE replies, stable conversations, and attachments
- Registering and verifying signed webhooks, delivery retries, and authenticated custom replies
- Using the V1 REST API with enabled-channel filtering, cursor pagination, and queued sends
- HelloMinds OAuth client/callback setup, recent history sync, and session removal
- Common troubleshooting

Read the [operator skill](whatsupmind-operator-manual/SKILL.md) and its
[API reference](whatsupmind-operator-manual/references/api.md).

## Source alignment

Checked against `whatsupmind-gh` **main**, commit
`ddf6113c13756e01a20cdb7839633268408097ec` (2026-10-13).
The implementation is authoritative where older application docs differ.
Current integrations use unified HelloMinds OAuth login, alias-scoped SSE for
Mind replies, and authenticated V1 sends for custom replies. Builder-key setup,
direct social/password login, and legacy reply-token/email-reply workflows are obsolete.

`whatsupmind-operator-manual/SKILL.md` is the canonical installable skill.
The repository follows the Agent Skills layout supported by the Skills CLI:

```text
README.md
skills.sh.json
whatsupmind-operator-manual/
├── SKILL.md
└── references/
    └── api.md
```

Validate discovery locally without installing:

```bash
npx skills add . --list
```

When updating, check the source files listed in the skill, update its API
reference, and advance the recorded source revision in both places. Verify
`origin/main` first: a local checkout on the legacy `old` branch is not the
source for the current product.

## License

MIT
