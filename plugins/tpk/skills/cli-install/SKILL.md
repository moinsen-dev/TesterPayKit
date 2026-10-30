---
name: cli-install
description: >
  Install the `tpk` TesterPayKit CLI globally from pub.dev so it is on
  PATH and usable from any directory. Trigger when the user says
  "install tpk", "install TesterPayKit CLI", "set up tpk", "global tpk",
  or asks how to make `tpk` reachable from any terminal. Also trigger on
  `command not found: tpk`. Installs the `tpk_cli` package (the `tpk`
  CLI, plus `tpk_mcp`, a local read-only MCP server).
---

# Install the tpk CLI globally

The tpk CLI is the agentic entry-point to the TesterPayKit ecosystem.
Every other testerpaykit skill assumes `tpk` is on PATH. It is published
on pub.dev as `tpk_cli` — no repo checkout needed.

## Steps

1. Check Dart is available:
   ```bash
   dart --version
   ```
   If missing: `brew install dart` on macOS, or install the Flutter SDK
   (which bundles Dart) — see dart.dev/get-dart.

2. Install from pub.dev:
   ```bash
   dart pub global activate tpk_cli
   ```
   This puts both `tpk` and `tpk_mcp` into `~/.pub-cache/bin/`.

3. Verify:
   ```bash
   which tpk
   tpk --help
   ```
   If `which tpk` returns nothing, `~/.pub-cache/bin` is not on PATH —
   add this to the user's shell rc and re-source:
   ```bash
   export PATH="$HOME/.pub-cache/bin:$PATH"
   ```

## What you get

- `tpk` — coordinator CLI (login, project, bug, session, job, tester,
  payout-rule). Every command supports `--json` for agentic use.
- `tpk_mcp` — local read-only MCP server (sessions, bugs, bug context,
  tester queue) using the `tpk login` credentials. This plugin no longer
  registers it; it uses the remote `testerpaykit` MCP server instead
  (`/tpk:connect`). Register `tpk_mcp` yourself only if you need it:
  `claude mcp add tpk -- tpk_mcp`.

## Troubleshooting

- **`dart: command not found`** — Dart SDK not installed (see step 1).
- **Stale binary from an earlier source-checkout install** — older
  installs compiled `tpk` from a git checkout into `~/.pub-cache/bin`.
  A leftover native binary shadows the pub.dev wrapper. Clean slate:
  ```bash
  rm -f ~/.pub-cache/bin/tpk ~/.pub-cache/bin/tpk_mcp
  dart pub global activate tpk_cli
  ```
- **`tpk` runs but talks to the wrong server** — check `tpk env current`.
  Fresh installs default to the production API
  (`https://api.testerpaykit.com`); that is what customers want.
