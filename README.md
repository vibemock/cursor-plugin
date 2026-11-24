# VibeMock Cursor Plugin

[![npm version](https://img.shields.io/npm/v/%40vibemock%2Fcli?label=%40vibemock%2Fcli&color=0ea5e9)](https://www.npmjs.com/package/@vibemock/cli)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-10b981.svg)](./LICENSE)
[![Cursor Plugin](https://img.shields.io/badge/Cursor-Plugin%20%2B%20MCP-0f172a)](https://vibemock.bid)

Official Cursor plugin for [VibeMock](https://vibemock.bid) — simulate 3rd-party APIs (compatible with Stripe, GitHub, Svix, Creem, Lemon Squeezy, Clerk, and custom HMAC routes) and deliver protocol-accurate signed webhooks directly to `localhost` without local tunnels or port forwarding.

## Why VibeMock for Cursor?

| Approach | Signature Accuracy | Tunnel / Daemon | Edge-Case Control |
| :--- | :--- | :--- | :--- |
| **Manual `curl` mocks** | Fake/skipped signatures — real signature middleware stays untested | None | Manual JSON edits per test |
| **Local tunnel tools** | Depends on external sandbox | Requires background daemon & changing URLs | Hard to reproduce `sync_fail` / `callback_drop` |
| **VibeMock Cursor Plugin** | **Protocol-accurate** (`Stripe-Signature`, `X-Hub-Signature-256`, Svix, HMAC-SHA256) | **Zero tunnels, zero background daemons** (`local://cli`) | **One-command switch**: `success`, `sync_fail`, `callback_drop` |

## What's Included

- **MCP Server (`mcp.json`)**: Runs `npx -y @vibemock/cli@latest mcp` over stdio with 10 structured tools:
  - `vibemock_whoami` — inspect authenticated identity, plan quotas, and daily usage
  - `vibemock_templates_list` — list built-in vendor templates
  - `vibemock_init_project` — create project-scoped `.vibemock.json` (never stores tokens)
  - `vibemock_apps_create_from_template` — bootstrap Stripe, Creem, Lemon Squeezy, GitHub, or Clerk mock apps
  - `vibemock_env_set` — switch scenario (`success`, `sync_fail`, `callback_drop`) and callback mode
  - `vibemock_mock_url_create` — issue short-lived, zero-header Mock URLs
  - `vibemock_forward_once` — pull pending cloud-signed callbacks to `localhost` synchronously
  - `vibemock_trigger_webhook` — synthesize and forward inbound-only signed webhooks in one shot
  - `vibemock_wait_record` — poll until callback reaches terminal state
  - `vibemock_records_list` — inspect captured inbound/outbound payloads and status
- **Agent Skill (`skills/vibemock/SKILL.md`)**: Multi-project & zero-background-process self-test loop.
- **Cursor Rule (`rules/vibemock-webhooks.mdc`)**: Context-aware guidance when editing webhook, billing, or callback files.
- **Slash Command (`commands/test-webhooks.md`)**: `/test-webhooks` end-to-end verification checklist.

## Prerequisites

1. Sign in at [console.vibemock.bid](https://console.vibemock.bid) (free plan available) and issue an **Inspect API Token** (`vmk_ins_...`) under **My Tokens**.
2. Save the token locally (never commit it to any repository):
   ```sh
   npx -y @vibemock/cli@latest config set --token <your_vmk_ins_token>
   ```
   Or export `VIBEMOCK_TOKEN` in your shell environment.

## Quick Start

1. Open your project in Cursor and ensure the `vibemock` MCP server is enabled.
2. Run `/test-webhooks` in Cursor Chat, or paste the example prompt below.
3. For manual or team installation without the Marketplace, clone this repository and add the folder in Cursor via **Settings → Plugins**.

### Example Cursor Prompt

```text
Use VibeMock to initialize .vibemock.json for my local webhook server on http://localhost:3000/api/webhooks/stripe,
create a stripe-checkout mock app, issue a short-lived Mock URL, and run the full verification matrix
(success, sync_fail, and callback_drop) using synthetic test data until all signature checks pass.
```

## Security & Privacy

- Use synthetic test payloads and fake signing secrets only — never send production secrets or real customer PII.
- Inspect API tokens (`vmk_ins_...`) are stored locally in `~/.config/vibemock/config.json` or `VIBEMOCK_TOKEN` and are never committed to project repositories.

---

*VibeMock is an independent developer tool and is not affiliated with Stripe, GitHub, Svix, Creem, Lemon Squeezy, or Clerk.*
