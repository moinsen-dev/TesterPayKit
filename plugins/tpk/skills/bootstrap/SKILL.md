---
name: bootstrap
description: >
  Bootstrap a brand-new TesterPayKit project end-to-end from inside a
  Claude Code session — log in the coordinator account (device-auth via
  browser), create the project (capturing its UUID), create a payout
  rule so testers actually earn money, and emit the env-var lines the
  host app needs. Trigger when the user says "set up TPK for this
  project", "create a TPK project", "bootstrap TesterPayKit", "onboard
  this app to TesterPayKit", or asks how to wire a fresh app into TPK.
  This is the canonical "first-touch" skill — runs once per host app.
  For an already-bootstrapped project the user wants the `status` or
  `integrate` skill instead.
---

# Bootstrap a TesterPayKit project

Goal: from zero to "the host app can submit bug reports to
api.testerpaykit.com — and testers get paid for them" in a handful of
CLI calls.

## Prerequisites

- `tpk` CLI installed (`which tpk` returns a path). If missing, run
  the `cli-install` skill first.
- API reachable: `curl -s https://api.testerpaykit.com/health` returns
  `{"status":"healthy",...}`. If not, surface that to the user and
  stop — there is nothing to bootstrap against.
- A coordinator account. Sign-up happens in the web console at
  **https://app.testerpaykit.com** (magic-link, passwordless). The CLI
  cannot create the account for a browser-first user — do that signup
  once, then come back here.

## Steps

1. **Check current login state.**
   ```bash
   tpk --json whoami 2>&1
   ```
   - On `not logged in` → step 2.
   - On a valid email payload → step 3.

2. **Log in (device-auth is the primary path).**
   ```bash
   tpk login
   ```
   This opens the browser to a verification page on the web console;
   the user approves the CLI session there (they must be signed in to
   app.testerpaykit.com via magic link). The CLI polls until approved
   and stores the JWT. Coordinator accounts are **invite-only** — if the
   user has none, stop and point them to business@moinsen.dev.

   There is no password login (removed in tpk_cli 0.3.0; the API dropped
   it with the magic-link migration). Non-interactive / CI runs pass a
   stored JWT with the global `--token` flag — never log it.

3. **Pick a project bundle-id.**
   For a Flutter app:
   - iOS:   `awk -F'= ' '/PRODUCT_BUNDLE_IDENTIFIER/ {print $2}' ios/Runner.xcodeproj/project.pbxproj | head -1`
   - Android: `grep applicationId android/app/build.gradle*`
   For a non-Flutter host, ask the user for a reverse-DNS bundle id.

4. **Check if a project already exists for that bundle-id.**
   ```bash
   tpk --json project get --bundle-id <bundle-id> 2>&1
   ```
   - On `{"id":"...", ...}` → reuse, skip step 5.
   - On an error → step 5.

5. **Create the project.**
   ```bash
   tpk --json project create \
     --name "<HumanName>" \
     --bundle-id <bundle-id> \
     --description "<one-liner>"
   ```
   Capture the `id` field — that's the project UUID.

6. **Create a payout rule — REQUIRED for paid testing.**
   Without an active payout rule the server computes €0 for every
   tester submission. "SDK integrated" does not mean "testers get
   paid" until this exists:
   ```bash
   tpk --json payout-rule create <project-uuid> \
     --name "Standard" \
     --base-amount 5.00 \
     --per-minute-rate 0.25 \
     --currency EUR \
     --max-payout 25.00
   ```
   Ask the user for the amounts — they are the coordinator's budget
   decision, not a default to guess. `--upsert` replaces an existing
   active rule (without it the server returns 409 if one exists).
   Verify with `tpk payout-rule list`.

7. **(Optional) Enable recruiting.**
   If the user wants testers to find and apply to the project via the
   TPK job board, recruiting is switched on in the web console
   (project → Recruiting tab on app.testerpaykit.com). Point the user
   there; check the pipeline later with `tpk job list` and
   `tpk job applications <project-uuid>`.

8. **Emit the wiring snippet.**
   Print to the user:
   ```
   TPK project created: <name> (<bundle-id>)
   UUID: <uuid>
   Payout rule: <name> — base <x> + <y>/min (<currency>)

   Pass this UUID to your app's TPK SDK — the `integrate` skill wires
   it per platform:
     Flutter:  flutter run --dart-define=TPK_PROJECT_ID=<uuid>
     Expo:     EXPO_PUBLIC_TPK_PROJECT_ID=<uuid>
     Web:      NEXT_PUBLIC_TPK_PROJECT_ID=<uuid>  (or VITE_TPK_PROJECT_ID)
   ```

9. **Hand off** to the `integrate` skill if the host app doesn't yet
   have a bridge file — that one handles the actual SDK wiring.

## What this skill does NOT do

- Doesn't push to git. Wiring changes are surfaced to the user; they
  decide when to commit.
- Doesn't install the `tpk` CLI — see `cli-install`.
- Doesn't create the coordinator account — that's the web-console
  signup (magic-link) at app.testerpaykit.com.
