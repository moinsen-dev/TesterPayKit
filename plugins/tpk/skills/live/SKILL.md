---
name: live
description: >
  See and drive the developer's own app on a real device from this Claude
  Code session, and keep a checklist of what testers should test. Trigger
  when the user says "look at my phone", "schau aufs Handy", "what does the
  screen show", "I sent you a note from the app", "tap X on my device",
  "walk through the app", "what should testers test", "make a checklist",
  "Checkliste", "how did the checklist go", or wants to switch on the live
  link in a Flutter, Expo, Web or Swift app. Needs the testerpaykit MCP
  server (see /tpk:connect) and SDK 1.2.0 or newer.
---

# Live link and checklist

The TesterPayKit SDKs (Flutter, Expo, Web, Swift — all 1.2.0) can connect a
running app to this session. Everything goes through the remote
`testerpaykit` MCP server; nothing runs on the device without the
developer switching it on.

## Switch it on in the app (once, in a dev / TestFlight build)

- Flutter: `TesterPayKitWrapper(enableLiveLink: true, navigatorKey: …)`
- Expo: `<TesterPayKitWrapper enableLiveLink …>`
- Web (React): `enableLiveLink` on `TpkTesterLauncher` / `TpkTesterHub`
- Swift: `TesterPayKitConfig(…, enableLiveLink: true)`

Optional app state for Claude: `liveStateProvider` (Flutter:
`TesterPayKit.instance.liveStateProvider = () => {...}`, Expo:
`tpk.liveStateProvider`, Swift: `TesterPayKit.liveStateProvider`). JSON
only, no secrets.

In the app the developer then opens the TPK button and taps **Connect to
Claude** ("Mit Claude verbinden"). The link holds while the app is in front.

## Look at the screen

- `list_live_devices` — who is connected; also reports `unread_notes`.
- `device_snapshot` — screenshot (as an image) plus every readable text
  and field value with positions in points, the current screen, recent
  navigation and breadcrumbs, the app state and SDK logs. Pass `device_id`
  only when several devices are connected.
- Notes the developer sent with **Send to Claude**: `list_device_notes`,
  then `get_device_note` (screenshot + screen contents; marks it read).
  When `unread_notes` > 0, read them first — they say what the developer saw.

## Drive the app (only the developer's own device)

`device_tap` (by visible `text` from the snapshot, or `x`/`y` in points),
`device_type` (`text`, optional `into` field label, `submit`),
`device_scroll` (`direction`: down reveals what is below), `device_back`.
Each returns the screen afterwards. The device refuses destructive actions
(delete account, pay, sign out) and TPK's own button. Limits: on iOS,
plain SwiftUI buttons need a loaded accessibility tree; Expo calls
`onPress` directly (no gesture-handler-only buttons).

## Checklist — what testers should test

- `get_checklist` / `set_checklist_task` (title, description, 3–8 concrete
  steps, each ending in something visible on screen) /
  `remove_checklist_task`. `project` = project id or bundle id.
- Testers see it in the app under **What to test** ("Was testen?") and give
  ✓/✗ plus a note per step; no assignment needed.
- `get_checklist_progress` — per task the latest human and AI pass with
  each step's result and note.
- To check a flow yourself: read the checklist, then walk the steps with
  `device_tap` / `device_type` and `device_snapshot`.

## After a fix

Marking a bug fixed (console or `tpk`) without naming a retester queues an
AI retest of the bug's acceptance steps — only an app at or above the fixed
version picks it up (Flutter and Expo, AI tester enabled). Passed →
verified; failed → reopened with the failing step.
