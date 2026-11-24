---
name: vibemock-test
description: Run an end-to-end VibeMock webhook and 3rd-party API verification loop (happy path, sync_fail, and callback_drop) against localhost.
---

# Test Webhooks End-to-End with VibeMock

Run a complete webhook & 3rd-party API verification loop for the current project:

1. Check configuration using `vibemock_whoami` (or `vibemock whoami --json`). If no `.vibemock.json` exists, initialize one with `vibemock_init_project` (using `local://cli` forwarding to the local server's webhook port).
2. Ensure an app and short-lived Mock URL exist (`vibemock_apps_create_from_template` / `vibemock_mock_url_create`), and verify the app under test points its 3rd-party `BASE_URL` at that Mock URL.
3. Run the verification matrix using synthetic test data only:
   - **Happy path (`success`)**: trigger the local endpoint (or `vibemock_trigger_webhook` for inbound-only hooks), run `vibemock_forward_once`, verify signature verification and `2xx` handler response, and assert `callback_status === "sent"` via `vibemock_wait_record`.
   - **Sync failure (`sync_fail`)**: switch scenario to `sync_fail`, trigger, and verify clean error handling with `callback_status === "skipped"`.
   - **Dropped webhook (`callback_drop`)**: switch scenario to `callback_drop`, trigger, and verify fallback/reconciliation behavior.
4. Restore scenario to `success`, fix any handler or signature bugs found, and report the final verification table.
