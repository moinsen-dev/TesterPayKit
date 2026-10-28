---
name: integrate
description: >
  Wire the TesterPayKit SDK into an existing app so it captures tester
  sessions and bug reports (shake-to-report, floating feedback button,
  post-crash dialog) and posts them to a TPK project — fail-safe, never
  blocking or crashing the host. Supports four SDK variants: Flutter
  (pub.dev `testerpaykit`, by default with `moinsen_runapp`), Expo /
  React Native (`@testerpaykit/expo`), Web / TypeScript — React,
  Next.js, Vite (`@testerpaykit/react` + `@testerpaykit/sdk`) — and
  native Swift for iOS/macOS (`TesterPayKit` Swift Package with the
  built-in SwiftUI tester experience; early access). Trigger when the
  user says "integrate TesterPayKit", "add TPK to this app", "wire up
  TPK", "set up shake-to-report", or asks how to use a TPK project
  they created — in a Flutter, Expo, React-Native, web, or native
  Apple codebase. Detects the platform, then follows the matching
  reference file. Run after `bootstrap` so the project UUID exists.
---

# Integrate TesterPayKit into an app

Goal: an existing app starts capturing tester sessions and bug reports
and posting them to the user's TPK project — without TPK ever being able
to crash or block the host app. The exact deps, file template, and
wiring differ per SDK; this skill detects the platform and hands off to
the right reference.

## Step 0 — detect the platform, then read ONE reference

Run these checks from the app root and load only the matching file:

```bash
# Flutter?
test -f pubspec.yaml && echo "FLUTTER"

# Expo / React Native?  (package.json has an `expo` dependency)
test -f package.json && node -e "process.exit(require('./package.json').dependencies?.expo?0:1)" \
  && echo "EXPO"

# Web / TypeScript?  (web bundler present, NO expo)
test -f package.json && node -e "const p=require('./package.json');const d={...p.dependencies,...p.devDependencies};process.exit(!d.expo&&(d.next||d.vite||d['react-scripts'])?0:1)" \
  && echo "WEB"

# Native Swift?  (SwiftPM or Xcode project, no Flutter/RN layer)
{ test -f Package.swift || ls *.xcodeproj *.xcworkspace >/dev/null 2>&1; } \
  && ! test -f pubspec.yaml && ! test -f package.json && echo "SWIFT"
```

| Detected | Read |
|---|---|
| `pubspec.yaml` present | **`references/flutter.md`** |
| `package.json` with an `expo` dep (or `app.json` / `app.config.*`) | **`references/expo.md`** |
| `package.json` with `next` / `vite` / `react-scripts` and **no** `expo` | **`references/web.md`** |
| `Package.swift` or `*.xcodeproj` / `*.xcworkspace`, no Flutter/RN layer | **`references/swift.md`** |
| ambiguous or none of the above | ask the user which target (Flutter / Expo / Web / Swift), then read that reference |

Then follow that reference end-to-end — it has the platform-correct
package names, the fail-safe bridge/init template, the mount/wiring, the
tester-onboarding path, and the verification commands.

## The four invariants (true on EVERY platform)

Whatever the reference says, these must hold — they are the reason the
integration is safe to ship:

1. **Fail-safe / non-blocking init.** TPK init runs inside a try/catch
   and must never throw into the host or block first paint/render. If it
   is gated off or init fails, the host runs clean and the SDK is inert.
2. **Build-flag / env gating.** Capture only in debug/dev builds or when
   an explicit beta flag is set — `--dart-define=<APP>_TPK=true`
   (Flutter), `EXPO_PUBLIC_TPK_ENABLED` (Expo),
   `NEXT_PUBLIC_TPK_ENABLED` / `VITE_TPK_ENABLED` (Web), a DEBUG
   compilation condition or env gate (Swift). A store/prod release
   without the flag ships the SDK inert: no capture, no API contact.
3. **The project UUID comes from `bootstrap`.** Never invent it. If the
   user doesn't have one, run the `bootstrap` skill first.
4. **Verify the host still boots and doesn't hang** with TPK enabled —
   including with the network down — before declaring done.

## Shared facts (all platforms)

- **Tester onboarding = invite/tester code → anonymous JWT.** The
  coordinator issues codes (TPK console / recruiting). The SDK trades
  `{projectId, code}` for a token; dev/CI can hardcode a code to skip
  the in-app screen.
- **Integration ≠ payout.** The server pays €0 until the project has an
  active payout rule (`tpk payout-rule list` / `bootstrap` step 6).
- **Screen/route tracking drives engaged-minutes → payouts.** Custom
  routers (GoRouter, React Navigation, Next/React Router) don't fire the
  built-in observer — call `trackScreenView('<screen>')` on route
  changes, or testers get underpaid.
- **Web has one extra gotcha:** the app's origin must be whitelisted on
  the project (`tpk project set-origins`) or the browser blocks every
  call with CORS. See `references/web.md`.
- **This skill does NOT auto-commit.** The user reviews the diff and
  decides.
