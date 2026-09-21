---
description: Configure PayPal plugin — verify credentials, test MCP server connection, and show active environment
argument-hint: [mode — e.g. "refresh", "status"]
allowed-tools: Read, Bash, mcp__paypal-sandbox__*
---

# PayPal Setup

Walk the user through PayPal plugin configuration for Claude Code. Verify everything works before they use other commands.

## Workflow

### Step 1: Check MCP Connection

Three checks, sandbox-first:

1. **Is the sandbox MCP connected?** Count tools whose name matches `mcp__paypal-sandbox__*` in this session. If any are present, run one lightweight probe (`mcp__paypal-sandbox__list_invoices` with `page_size: 1`). Report the tool count from the session.
2. **Does the project `.env` have sandbox credentials?** `grep -q "^PAYPAL_CLIENT_ID=.\+" .env && grep -q "^PAYPAL_CLIENT_SECRET=.\+" .env` (existence only — never read or print the values).
3. **Is the legacy sandbox token set in `~/.claude/settings.json`?** Read the file and check whether the `"env"` block has a non-empty `"PAYPAL_SANDBOX_ACCESS_TOKEN"`. This is the older, pre-auto-refresh path; it still works but isn't recommended for new setups.

#### Report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PayPal Plugin Setup
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  [✓] Sandbox MCP                — connected (31 tools)
  [✓] Sandbox credentials        — in .env (auto-refreshing)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If the MCP isn't connected OR the probe returns an error (401, SSE header invalid, etc.), report `needs setup` and go to Step 2. If the MCP is connected but tool count looks low (less than 10), note it briefly and proceed to Step 2 to help the user upgrade.

```
  [!] Sandbox MCP                — needs setup
  [!] Sandbox credentials        — not in .env
  [!] Legacy token in settings   — not set
```

**If MCP tools aren't loaded at all** (no `mcp__paypal-sandbox__*` tools in the session and `/mcp` doesn't list `paypal-sandbox`): the plugin isn't installed correctly. Fix:

- Marketplace: `/plugin marketplace add paypal/AI-Toolkit` then `/plugin install paypal@paypal-ai-toolkit`
- Local: `claude --plugin-dir /path/to/AI-Toolkit`

Set up the sandbox connection using `PAYPAL_CLIENT_ID`/`PAYPAL_CLIENT_SECRET` in the project's `.env` (via Step 3 below) — this is the recommended path: the plugin's `headersHelper` reads them directly from `.env` and mints/auto-refreshes the access token in the background, so there's no manual token regeneration and no separate credentials file. The legacy env-var route (`PAYPAL_SANDBOX_ACCESS_TOKEN` in `~/.claude/settings.json`) still works and takes priority if `.env` credentials aren't found.

The `SSE error: Header has invalid value` failure mode on the legacy path means the env var is set but contains whitespace/newlines (usually from a multi-line `~/.zshrc` export). Fix: regenerate, put on a single line in `settings.json`, or switch to `.env` credentials via Step 3.

### Step 2: Diagnose why a set token still fails

Reach this step only when Step 1 reports credentials are in `.env` (or the legacy token is set) but the MCP server still fails to connect (probe returns 401, SSE error, or no tools loaded).

Likely causes, in order of frequency:

1. **`.env` credentials are wrong or the sandbox app was deactivated.** `headersHelper` silently falls back to no dynamic header if it can't mint a token, so if there's no legacy token either, the connection has no valid Authorization header at all. → Step 3 to double check the values.
2. **Legacy token expired.** Only applies if the user is on the legacy `settings.json` path — PayPal access tokens last up to 8 hours. `.env`-based credentials (via `headersHelper`) refresh automatically and shouldn't hit this. → Step 3 to migrate to `.env`, or regenerate the legacy token.
3. **Legacy token has stray whitespace or a newline.** Usually from a copy-paste that picked up the surrounding quote characters, or from a multi-line `~/.zshrc` export that was migrated to `settings.json` without flattening. JSON should reject this, but if it survived, the `Authorization` header will be rejected by Node's HTTP layer (`SSE error: Header has invalid value`). → Open `settings.json`, confirm the value is a single line with no whitespace, or switch to Step 3.
4. **Gateway rate-limited (`HTTP 429`).** You've been reconnecting too often during testing. Auth is fine — just wait 1–2 minutes. Do not regenerate.
5. **Network issue.** Behind a corporate VPN/proxy. Check connectivity to `mcp.sandbox.paypal.com` and `api-m.sandbox.paypal.com`.

If none of the above resolve it, go to Step 3 and set up credentials from scratch.

**If neither `.env` credentials nor the legacy token were found in Step 1:** skip Step 2 and go straight to Step 3.

### Step 3: Credential Setup

Guide the user through adding sandbox credentials to their project's `.env` so the plugin can mint and auto-refresh tokens itself.

#### Sandbox Setup

```
Let's set up your PayPal sandbox credentials.

1. Go to: https://developer.paypal.com/dashboard/applications/sandbox
2. Create an app (or use the default "My Testing Application")
3. Copy your Client ID and Client Secret
4. Add them to your project's .env file (create one if it doesn't exist):

  PAYPAL_CLIENT_ID=your_client_id
  PAYPAL_CLIENT_SECRET=your_client_secret

The plugin reads these directly from .env and mints/refreshes the sandbox
access token automatically — no restart needed after the token expires.
Restart Claude Code once now so the MCP server picks up the new .env values.

Never paste your Client Secret into this chat — add it directly to .env
in your editor.
```

Wait for the user to confirm they've added the values and restarted. Then re-verify by attempting an MCP tool call.

**If 401 / Unauthorized:** "Credentials were rejected. Double check the Client ID and Client Secret in the PayPal Developer Dashboard, then update .env and restart Claude Code."

**If timeout / network error:** "Check your network connection. If you're behind a corporate proxy or VPN, you may need to allowlist `mcp.sandbox.paypal.com` and `api-m.sandbox.paypal.com`."

**Legacy path (not recommended for new setups):** if a user specifically wants the old static-token flow instead, they can still set `PAYPAL_SANDBOX_ACCESS_TOKEN` in `~/.claude/settings.json` by generating a token via `curl -X POST https://api-m.sandbox.paypal.com/v1/oauth2/token -u "CLIENT_ID:CLIENT_SECRET" -d "grant_type=client_credentials"` and pasting the `access_token` value in as a single line, then fully restarting Claude Code. It works, but the token must be regenerated manually roughly every 8 hours.

Note: `.paypal_token_cache.json` (the derived, short-lived Bearer token the plugin caches locally) should be added to the project's `.gitignore` — it's already covered if `.env` is gitignored the standard way, but call it out if the project doesn't gitignore `.env*` patterns broadly.

### Step 3.5: Project Environment Check

After MCP verification, scan the user's project's `.env` for the same `PAYPAL_CLIENT_ID`/`PAYPAL_CLIENT_SECRET` variables. These now do double duty: the plugin's `headersHelper` uses them for sandbox MCP auth (Step 3), and if the project also makes its own server-side PayPal REST API calls, its code reads the same variables directly. This step just confirms the project's `.env` is in good shape for both.

**Check for `.env` files:**

```bash
ls .env .env.sample .env.example 2>/dev/null
```

**Check for v6 integration signals:**

```bash
grep -rl "web-sdk/v6/core\|createInstance\|paypal-payments" --include="*.js" --include="*.jsx" --include="*.ts" --include="*.tsx" --include="*.html" . 2>/dev/null | head -5
```

**Logic:**

- If `.env.sample` or `.env.example` exists but no `.env` → flag it:

  ```
  [!] Project Environment
      Found .env.sample but no .env file.
      Copy it and fill in your credentials:

        cp .env.sample .env

      Required variables for v6 server endpoints (also used by the plugin's
      sandbox MCP auth):
        PAYPAL_CLIENT_ID=your_client_id
        PAYPAL_CLIENT_SECRET=your_client_secret
        PAYPAL_ENVIRONMENT=sandbox
  ```

- If `.env` exists → check it has `PAYPAL_CLIENT_ID` and `PAYPAL_CLIENT_SECRET` set (non-empty), without ever printing the values:

  ```bash
  grep -q "^PAYPAL_CLIENT_ID=.\+" .env && echo "PAYPAL_CLIENT_ID: set" || echo "PAYPAL_CLIENT_ID: missing"
  grep -q "^PAYPAL_CLIENT_SECRET=.\+" .env && echo "PAYPAL_CLIENT_SECRET: set" || echo "PAYPAL_CLIENT_SECRET: missing"
  ```

  If missing → flag which keys are absent. Never echo, log, or otherwise surface the actual value of `.env` contents.

- If no `.env.sample` and no `.env` but v6 signals detected → flag it:

  ```
  [!] Project Environment
      v6 SDK integration detected but no .env file found.
      Your server endpoints need PAYPAL_CLIENT_ID and PAYPAL_CLIENT_SECRET.
      Create a .env file in the project root with these variables.
  ```

- If no v6 signals and no `.env.sample` → skip silently.

Include the project environment status in the Step 4 summary output.

### Step 4: Environment Summary

After successful connection, present a summary:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PayPal Plugin — Ready
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  [✓] Sandbox MCP                — connected (31 tools)
  [✓] Sandbox credentials        — in .env (auto-refreshing)

  Your project
  [✓] .env — PAYPAL_CLIENT_ID and PAYPAL_CLIENT_SECRET set
       — or —
  [!] .env.sample found but no .env (copy it and fill in credentials)
       — or —
  [ ] No server-side PayPal code detected — skipped

  Tokens refresh automatically. No restart needed.

  Useful commands
  /paypal:doctor [symptom]       — Scan your code for integration issues
  /paypal:explain-error <code>   — Explain a PayPal error code
  /paypal:sandbox [topic]        — Sandbox setup reference
  /paypal:test-accounts [topic]  — Test scenarios and accounts

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Step 5: Suggest Next Steps

Detect PayPal code in the project root:

```bash
grep -rl "paypal\|@paypal/\|api-m.paypal.com" --include="*.js" --include="*.jsx" --include="*.ts" --include="*.tsx" --include="*.html" --include="*.py" --include="*.java" --include="*.rb" --include="*.php" . 2>/dev/null | head -5
```

**If the project has PayPal-related code:**

```
Your project has PayPal integration code. Try:
  /paypal:doctor — Scan your integration for issues
```

**If the project has no PayPal code:**

```
No PayPal code detected in this project. Try:
  - Ask me to "create a PayPal checkout flow" and I'll help you build one
  - /paypal:sandbox — Learn about the sandbox environment
  - /paypal:test-accounts — See test scenarios and test card numbers
```

**If the user already has everything working:**

```
You're all set. Try asking me to:
  - Create an order or capture a payment (via MCP tools)
  - Build a checkout page with the PayPal JS SDK
  - Set up webhook handling for your server
```

## Error Reference (LLM-facing)

| Error                                                | Likely Cause                                                                                                                                                  | Fix                                                                                                                                            |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| No `mcp__paypal-sandbox__*` tools in session         | Plugin not loaded                                                                                                                                             | Install via marketplace or restart: `claude --plugin-dir /path/to/AI-Toolkit`                                                                  |
| 401 Unauthorized                                     | `.env` credentials or legacy token rejected                                                                                                                  | Re-run Step 3 to fix the `.env` values (or regenerate the legacy token), then restart Claude Code                                              |
| `SSE error: Header has invalid value`                | Legacy token in `settings.json` contains a newline/whitespace (multi-line `~/.zshrc` export)                                                                 | Move to `.env` credentials (Step 3), or fix the legacy value to a single line                                                                  |
| `SSE error: Non-200 status code (429)` or `HTTP 429` | PayPal's hosted gateway is rate-limiting reconnects from the same client (commonly hit when toggling Enable/Authenticate/Reconnect repeatedly during testing) | Wait 1–2 minutes before retrying. Don't change credentials or `.mcp.json` — the auth is fine, the gateway just needs the request rate to cool down. |
| Timeout / connection refused                         | Network, proxy, or VPN blocking                                                                                                                               | Allowlist `mcp.sandbox.paypal.com` and `api-m.sandbox.paypal.com`                                                                              |
| 403 Forbidden                                        | Credentials valid but missing API permissions                                                                                                                | Check app permissions in the PayPal Developer Dashboard                                                                                        |
