---
name: connect
description: >
  Connect this Claude Code setup to the remote TesterPayKit MCP server so
  Claude can order tests with real human testers and read their results.
  Trigger when the user says "connect TesterPayKit", "tpk connect", "set up
  the TPK MCP", "order a test with real testers", "get real testers on
  this", or when the `testerpaykit` MCP server shows as failed / 401 /
  not connected. Also covers how a test order works and the money rule.
---

# Connect the TesterPayKit MCP server

The plugin registers the MCP server `testerpaykit` at
`https://api.testerpaykit.com/mcp`. It authenticates with a personal access
token read from the environment variable `TPK_API_TOKEN`. Without it the
server stays disconnected (HTTP 401) — nothing else breaks.

## Steps

1. **Token.** The user creates it themselves — never ask them to paste it
   into the chat:
   app.testerpaykit.com → Profile → **Access tokens for Claude Code** →
   name it (e.g. the project) → **Create token**. It starts with
   `tpk_pat_` and is shown once.

2. **Make it available to Claude Code.** Tell the user to add this line to
   their shell profile (`~/.zshrc` or `~/.bashrc`) with their token, then
   open a new terminal and restart Claude Code:
   ```bash
   export TPK_API_TOKEN=tpk_pat_…
   ```
   Never put the token into a `claude mcp add --header` command (it lands in
   plaintext in the Claude config). If the user keeps secrets in a store,
   register the server with a `headersHelper` that prints
   `{"Authorization":"Bearer <token>"}` from that store instead.

3. **Check.** After the restart the tools `list_test_orders`,
   `create_test_order`, … are available. Call `list_test_orders`; an empty
   list is fine. A 401 means the token is missing, mistyped or revoked.

## How a test order works

1. `create_test_order` — app (`bundle_id`, or `app_url` for web apps),
   `platforms`, `tester_count`, `reward_per_tester_eur` (0 = unpaid
   community test) and `tasks` (title + steps). Free; the listing stays
   closed. Calling it again with the same bundle id reuses the project.
2. `publish_test_order` **costs money.** Called without `confirm_token`
   it changes nothing and returns the price (testers × reward) and a
   token valid for 15 minutes. Show the price to the user and wait for an
   explicit yes. Only then call it again with `confirm_token`. Never
   confirm on your own.
3. Applicants are accepted by the user in the console.
4. `get_test_order` for the status, `get_test_results` for what testers
   did and said (step verdicts with notes, would use / pay / recommend,
   bug reports), `list_bugs` / `get_bug` for details.

Every result carries `origin`: `"human"` for real people, `"ai"` for
TesterPayKit's own AI testers. `get_test_results` returns only human
results unless `include_ai` is true. Keep the two apart when you report —
an AI tester's finding is not evidence from a real user.
