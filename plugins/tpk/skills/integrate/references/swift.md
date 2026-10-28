# Integrate TesterPayKit — Swift (iOS / macOS) reference

Platform reference for the `integrate` skill. Read this when Step 0 of
`SKILL.md` detected a native Apple project (`Package.swift` or an
`*.xcodeproj` / `*.xcworkspace`, no Flutter/React-Native layer). The
four universal invariants in `SKILL.md` still apply — this file is the
Swift-specific recipe.

The Swift SDK is a native Swift Package (`TesterPayKit`, an actor) with
full behavioral parity to the Flutter reference: non-blocking init,
anonymous tester auth, durable offline outbox, crash auto-capture,
weekly earnings submission with server-verified integrity hash — plus a
complete built-in SwiftUI tester experience (floating feedback button,
bug report, voice input, screenshot annotation, tester dashboard,
session-end dialog incl. the demand-signal questions).

## Availability (be honest with the user)

The package is **not yet published** to a public registry. Today it is
consumed as a **local path dependency** from a checkout of the
`moinsen-infrastructure` monorepo (this is how Pumuckl, the first host,
integrates). If the user has no repo access, tell them the Swift SDK is
early-access — contact business@moinsen.dev — and stop; do not invent a
public URL.

```swift
// Package.swift
dependencies: [
    .package(path: "../moinsen-infrastructure/testerpaykit/sdks/swift"),
],
targets: [
    .executableTarget(
        name: "MyApp",
        dependencies: [
            .product(name: "TesterPayKit", package: "swift"),
        ]
    ),
]
```

XcodeGen users add the same path under `packages:` in `project.yml`.
Platforms: iOS 15+ / macOS 12+.

## Prerequisites

- TPK project UUID is known (run the `bootstrap` skill first).
- Decide the gating posture with the user: the SDK itself is
  release-safe, but the convention is DEBUG builds + explicitly
  flagged beta builds only (mirror the Flutter `MYAPP_TPK` pattern
  with a compilation condition or environment check).

## Steps

1. **Add the package** (see Availability above) and build once so SPM
   resolves it.

2. **Write a fail-safe service actor** — TPK must never crash or block
   the host. This is the pattern proven in Pumuckl:

   ```swift
   import Foundation
   @preconcurrency import TesterPayKit

   /// Fail-safe bridge to TesterPayKit. Every method swallows its own
   /// errors — an embedded testing SDK must never crash the host.
   actor TesterPayKitService {
       static let shared = TesterPayKitService()

       private static let projectId = "<UUID-from-bootstrap>"
       private static let testerCode = "<INVITE-CODE or dogfood code>"
       private static let apiBaseURL = URL(string: "https://api.testerpaykit.com")!

       nonisolated(unsafe) private var sdk: TesterPayKit?
       private init() {}

       /// Call once at launch. `initialize` returns fast and NEVER
       /// blocks on the network — token acquisition runs in the
       /// background (await `onTokenReady()` before the first
       /// authenticated read if you need one).
       func start() async {
           guard sdk == nil else { return }
           do {
               let instance = try await TesterPayKit.initialize(
                   config: TesterPayKitConfig(
                       projectId: Self.projectId,
                       testerCode: Self.testerCode,
                       apiBaseURL: Self.apiBaseURL
                   )
               )
               try await instance.startSession()
               await instance.installCrashHandler()
               sdk = instance
           } catch {
               NSLog("[TPK] init failed (non-fatal): %@", String(describing: error))
           }
       }

       /// End the session cleanly at quit — otherwise sessions leak as
       /// perpetually active. On macOS also call
       /// `ProcessInfo.processInfo.disableSuddenTermination()` at launch
       /// so applicationWillTerminate actually runs.
       func stop() async {
           await sdk?.onAppDetached()
       }
   }
   ```

   Wire `start()` in `applicationDidFinishLaunching` (AppKit) or the
   App `init` (SwiftUI), and `stop()` in `applicationWillTerminate` /
   `scenePhase == .background` as appropriate. On iOS also forward
   `onAppResumed()` / `onAppPaused()` from scene-phase changes — they
   drive the paid-time clock.

3. **Mount the tester UI.** Two options:

   **(a) One-liner — the built-in overlay** (floating draggable button
   + radial menu: report bug · feedback · dashboard · end session):
   ```swift
   ContentView().tpkFeedbackOverlay()
   ```

   **(b) À la carte** — mount individual surfaces where they fit the
   host (this is what a menu-bar app like Pumuckl does):
   - `TPKBugReportView` / `.tpkBugReportSheet(isPresented:)`
   - `TPKGeneralFeedbackDialog` / `.tpkGeneralFeedbackSheet(...)`
   - `TPKTesterDashboardView` — points/rank, weekly reports, retests,
     tasks, session details in one view
   - `TPKSessionEndDialog` / `.tpkSessionEndSheet(...)` — ends the
     session and asks the demand-signal questions (would you use /
     pay / recommend?)
   - `TPKOnboardingView` / `.tpkOnboardingSheet(...)` — welcome +
     invite-code login for runtime tester onboarding
   - `TPKVoiceBugInputView` / `.tpkVoiceBugSheet(...)` — spoken bug
     description, server-extracted into a reviewable draft (host owns
     the microphone permission string)
   - `TPKScreenshotAnnotationEditor` — draw on a screenshot before
     attaching it (`Screenshot.capturePNG()` +
     `submitBugReportWithScreenshot`)
   - `TPKMyBugsView`, `TPKNotificationsView`, `TPKRetestQueueView`,
     `TPKTaskTrackingPanel`, `TPKSessionDetailsView`
   - Shake-to-report on iOS: `.tpkShakeToReport()`

4. **Screen tracking.** SwiftUI: `.tpkScreen("ScreenName")` per screen.
   UIKit/AppKit: `TesterPayKit.trackScreen("ScreenName")` in
   `viewDidAppear`.

5. **Verify.**
   ```bash
   swift build && swift test   # host still builds
   ```
   Then run the app and check the TPK console (or `tpk session list`)
   for the new session. A bug report submitted while offline must
   appear after reconnect — that's the durable outbox doing its job.

## Swift-specific invariant notes

- `TesterPayKit` is an **actor**; all calls are `await`ed. UI views read
  `TesterPayKit.shared` themselves — they render an empty/neutral state
  when the SDK never came up (invariant 1: fail-safe).
- The offline outbox (invariant 3) persists across restarts and
  deduplicates via idempotency keys; 4xx responses park items as
  `failed_permanent` instead of retrying forever.
- Crash auto-capture (NSException + signals) reports on next launch;
  enable network capture only opt-in (`enableNetworkCapture()`).
