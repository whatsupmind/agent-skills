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
- Configuring HelloMinds Builder keys, account forwarding, per-channel Minds, and fallback selection
- Understanding group mention filtering, SSE replies, stable conversations, and attachments
- Registering and verifying signed webhooks, delivery retries, and authenticated custom replies
- Using the V1 REST API with enabled-channel filtering, cursor pagination, and queued sends
- OAuth login setup (Google, GitHub, X)
- Common troubleshooting

Read the [operator skill](whatsupmind-operator-manual/SKILL.md) and its
[API reference](whatsupmind-operator-manual/references/api.md).

## Source alignment

Checked against `whatsupmind-gh` commit
`d9a6d5f46af65d4140a1b74b7cb83397b5bd50e4` (2026-10-13).
The implementation is authoritative where older application docs differ.
Current integrations use SSE for Mind replies and authenticated V1 sends for
custom replies; legacy reply-token and email-reply instructions have been removed.

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
reference, and advance the recorded source revision in both places.

## License

MIT
