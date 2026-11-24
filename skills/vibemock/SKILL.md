---
name: vibemock
description: Use the VibeMock CLI to configure third-party API & Webhook mock environments, issue short-lived Mock URLs, wait for async callback completion, and inspect captured inbound payloads during local development and integration testing.
---

# VibeMock Agent Skill

Use `vibemock` whenever you need to test or debug code that integrates with mocked third-party APIs or asynchronous webhooks hosted on VibeMock.

> **Cursor / MCP note**: If the `vibemock` MCP server is active, prefer calling the structured `vibemock_*` MCP tools (`vibemock_whoami`, `vibemock_init_project`, `vibemock_apps_create_from_template`, `vibemock_env_set`, `vibemock_mock_url_create`, `vibemock_forward_once`, `vibemock_trigger_webhook`, `vibemock_wait_record`, `vibemock_records_list`) directly instead of shelling out.

## Prerequisites & Configuration

The CLI reads `~/.config/vibemock/config.json` automatically. If `api_url` is not configured, it defaults to `https://api.vibemock.bid`.

Check current configuration and identity first:

```sh
vibemock config show --json
vibemock whoami --json
```

If no token is configured yet, ask the user to issue an **Inspect Token** from the VibeMock Web Console and save it with:

```sh
vibemock config set --token <inspect_token>
# Optional: override default API URL (e.g. local wrangler dev)
vibemock config set --api-url http://api.localhost:8787
# Optional: set defaults to avoid repeating flags
vibemock config set --default-app <appid> --default-env <env>
```

> **Security rule**: Never print, log, or commit plaintext tokens into source files, test fixtures, or git history. Always use synthetic test data only.

## Multi-Project & Zero-Background-Process Best Practices

When working across multiple repositories on the same machine:

1. **Isolate per-project defaults with `.vibemock.json` (never store `token` inside a repo)**:
   - Run `vibemock init --app <appid> --env <env> --callback-url http://localhost:<port>/webhook --json` inside the project root, or pass `--app` and `--env` explicitly on every command.
   - Do **not** overwrite global `~/.config/vibemock/config.json` `default_app` / `default_env` when switching projects, or concurrent agents will collide.
2. **Reuse one App across projects via isolated `env` names**:
   - If multiple projects integrate the same vendor (e.g. Stripe), reuse one App (such as `stripe-shared`) and isolate each project with its own `env` (e.g. `proj-a`, `proj-b`).
3. **Zero-background-process sync loop (preferred over background daemons)**:
   - Because `local://cli` callbacks persist in VibeMock D1 until pulled, **trigger your local code first**, then run `vibemock forward --once --json` synchronously. No long-running background watcher is needed.
4. **Inbound-only webhooks (no preceding sync API call)**:
   - Use `vibemock trigger` to synthesize a protocol-signed webhook directly and forward it to localhost in one shot:
     ```sh
     vibemock trigger --app <appid> --env <env> --event invoice.paid --payload '{"id":"evt_1","type":"invoice.paid"}' --to http://localhost:3000/api/webhooks/stripe --json
     ```

## Standard Self-Test Loop

Always pass `--json` when invoking `vibemock` so output is structured and easy to assert.

### 1. Discover or bootstrap from official vendor templates

```sh
vibemock templates list --json
vibemock apps list --json
# Quickly bootstrap Stripe / Creem / Lemon Squeezy / GitHub / Clerk mocks:
vibemock apps create-from-template stripe-checkout --app stripe-01 --json
vibemock env list --app stripe-01 --json
```

Use your own private `env` name (for example `agent-local` or your personal env). Do **not** modify shared environments with `is_protected = 1`.

### 2. Configure the scenario & callback mode for `(appid, env)`

For **local development (no tunnel or domain verification required)**, set `--callback-url local://cli` and use `vibemock forward` to deliver cloud-signed webhooks directly to your `localhost` server:

```sh
vibemock config set --callback-url http://localhost:3000/api/webhooks/stripe
vibemock env set --app <appid> --env <env> --scenario success --callback-url local://cli --json

# After triggering your local code, forward pending cloud-signed callbacks to localhost (command --to overrides config):
vibemock forward --app <appid> --env <env> --to http://localhost:3000/api/webhooks/stripe --once --json
```

For **cloud direct callbacks (CI / staging)**, verify ownership of your callback domain first, then configure the HTTPS URL:

```sh
vibemock domains add staging.example.com --json
vibemock domains verify <domain_id> --method http_well_known --json
vibemock env set --app <appid> --env <env> \
  --scenario success \
  --callback-url https://staging.example.com/webhook \
  --sync-delay 0 \
  --callback-delay 500 \
  --json
```

Common scenarios:
- `success`: sync 2xx + async callback success (for `async` routes)
- `sync_fail`: sync 4xx/5xx rejection (automatically skips async callback -> `callback_status = "skipped"`)
- `async_fail` (alias `callback_fail`): sync 2xx + async callback failure payload
- `async_timeout` (alias `callback_drop`): sync 2xx + intentionally drops callback (`callback_status = "skipped"`)
- Custom scenario names defined in the app's route rules are also supported.

### 3. Issue a short-lived Mock URL (zero custom auth headers)

```sh
vibemock mock-url create --app <appid> --env <env> \
  --purpose "Local agent self-test" \
  --ttl 3600 \
  --json
```

Inject the returned `base_url` (`https://{env}--{appid}--{token}.<domain>`) into the service under test as its third-party `BASE_URL`. Do **not** add custom auth headers to mock traffic.

### 4. Trigger the code under test, then wait for completion

Prefer `vibemock wait` over manual sleep loops. It polls `/_inspect/records` until `callback_status` reaches a terminal state (`none`, `sent`, `failed`, or `skipped`):

```sh
vibemock wait --app <appid> --env <env> --biz-key <order_or_biz_id> --timeout 15000 --json
```

Or wait by `trace_id` (from response header `X-Mock-Trace-Id`):

```sh
vibemock wait --trace-id <trace_id> --timeout 15000 --json
```

### 5. Verify captured data & clean up

Inspect the returned `record`:
- `inbound_method`, `inbound_path`, `inbound_headers_json`, `inbound_query`, `inbound_body` (check `inbound_body_truncated`)
- `sync_status`, `sync_body`
- `callback_status` (`none` | `pending` | `sent` | `failed` | `skipped`)
- `callback_url`, `callback_payload`, `callback_response_status`, `callback_response_body`, `callback_attempts`

Clear synthetic test records when finished:

```sh
vibemock records clear --app <appid> --env <env> --biz-key <order_or_biz_id> --json
```

## Troubleshooting

- **`401 Unauthorized`**: Token expired/revoked or missing. Run `vibemock config show` and ask the user to refresh the Inspect Token.
- **`403 Forbidden` on `env set`**: The target `(appid, env)` is protected (`is_protected = 1`) or owned by another user. Switch to a personal `env` name.
- **`callback_status = "failed"` with SSRF error**: Cloud VibeMock blocks private/loopback callback URLs and requires `https:` plus a verified callback domain. For local development against cloud VibeMock, set `--callback-url local://cli` and run `vibemock forward` instead of exposing localhost. For CI or staging cloud callbacks, register and verify your domain via `vibemock domains add` and `vibemock domains verify`.
