# Integrate TesterPayKit — Flutter reference

Platform reference for the `integrate` skill. Read this when Step 0 of
`SKILL.md` detected a Flutter project (`pubspec.yaml` present). The four
universal invariants in `SKILL.md` still apply — this file is the
Flutter-specific recipe.

Default combo for native targets: the `testerpaykit` SDK **plus**
`moinsen_runapp` (the LLM debug bridge). Web targets skip
`moinsen_runapp` (it needs `dart:io`).

## Prerequisites

- Target app is a Flutter project (has `pubspec.yaml` + `lib/main.dart`).
- TPK project UUID is known. If unknown, run the `bootstrap` skill first.
- Native target (iOS / Android / desktop). `moinsen_runapp` uses
  `dart:io` and does NOT run on Flutter Web — for a web target, skip
  the moinsen_runapp parts and ask the user before proceeding.

## Steps

1. **Verify the project layout.**
   ```bash
   test -f pubspec.yaml && head -1 pubspec.yaml
   test -f lib/main.dart
   ```

2. **Detect the host's privacy / build-mode posture.**
   Read any project-level `CLAUDE.md` / `VISION.md`. Default gating:
   TPK runs in debug builds and in builds compiled with
   `--dart-define=<APP>_TPK=true` (intended for TestFlight / beta
   tracks). Store-release builds without the flag ship with the SDK
   inert — no capture, no API contact. Rename `<APP>_TPK` to a
   project-appropriate flag (e.g. `MYAPP_TPK`).

3. **Add the dependencies (plain pub.dev — no path deps, no overrides).**
   ```bash
   flutter pub add testerpaykit moinsen_runapp
   ```

4. **Write `lib/core/tpk/testerpaykit_bridge.dart`.**
   Template (verified against testerpaykit 0.4.x — `initialize` returns
   in <50 ms, token acquisition + uploads happen in the background via
   an offline-capable outbox):
   ```dart
   import 'package:flutter/foundation.dart';
   import 'package:testerpaykit/testerpaykit.dart';

   /// Project UUID from `tpk project create` (bootstrap skill).
   const String _projectId = String.fromEnvironment(
     'TPK_PROJECT_ID',
     defaultValue: '<UUID-from-bootstrap>',
   );
   const String _apiBaseUrl = 'https://api.testerpaykit.com';

   /// Build-time gate: debug builds always, release builds only with
   /// --dart-define=MYAPP_TPK=true (TestFlight / beta).
   const bool _enabledViaDartDefine = bool.fromEnvironment(
     'MYAPP_TPK',
     defaultValue: false,
   );
   bool get isTesterPayKitEnabled => kDebugMode || _enabledViaDartDefine;

   /// Dev/CI escape hatch: bake a tester code in at build time so the
   /// in-app invite-code screen is skipped.
   ///   flutter run --dart-define=TPK_TESTER_CODE=<code>
   const String _testerCode = String.fromEnvironment(
     'TPK_TESTER_CODE',
     defaultValue: '',
   );

   /// Null-safe handle for call sites that emit telemetry — no-ops via
   /// `?.` in builds where the SDK is absent.
   TesterPayKit? get currentTesterPayKit =>
       TesterPayKit.isInitialized ? TesterPayKit.instance : null;

   /// Initialise TPK. Fail-safe: returns null when gated off or when
   /// init throws — TPK must never take the host app down.
   Future<TesterPayKit?> initTesterPayKit() async {
     if (!isTesterPayKitEnabled) return null;
     if (_projectId.isEmpty || _projectId.startsWith('<')) return null;
     try {
       return await TesterPayKit.initialize(
         config: TesterPayKitConfig(
           projectId: _projectId,
           apiBaseUrl: _apiBaseUrl,
           // null → in-app invite-code onboarding (the normal tester
           // path); non-null → build-time auth (dev/CI).
           testerId: _testerCode.isEmpty ? null : _testerCode,
           enableTesterMode: true,
           // Upload crashes silently instead of asking the tester to
           // confirm on next launch — less friction for paid testers.
           crashAutoUpload: true,
           debugMode: kDebugMode,
         ),
       );
     } catch (e, st) {
       debugPrint('[tpk_bridge] init failed: $e\n$st');
       return null;
     }
   }
   ```
   Optional: set `bundleId: '<bundle-id>'` in the config when the app
   ships variants whose real applicationId differs from the TPK
   project's bundle-id — otherwise the SDK uses
   `PackageInfo.fromPlatform()` and reports can land mismatched.

5. **Wire `main.dart` — TPK INSIDE `moinsenRunApp(init:)`.**
   The order matters for two reasons: (a) if TPK init fails, moinsen
   catches it and the app still boots; (b) TPK's crash handler chains
   to the previously installed `FlutterError.onError`, so initialising
   TPK after moinsen means BOTH layers see every error (TPK files the
   bug, moinsen shows it in the dev overlay).
   ```dart
   import 'package:flutter/foundation.dart';
   import 'package:flutter/material.dart';
   import 'package:moinsen_runapp/moinsen_runapp.dart';
   import 'package:testerpaykit/testerpaykit.dart';

   import 'core/tpk/testerpaykit_bridge.dart';

   TesterPayKit? _tpk;
   final _navigatorKey = GlobalKey<NavigatorState>();

   void main() {
     moinsenRunApp(
       // Awaited before MyApp builds. If it throws, moinsen logs it
       // and the app starts anyway (_tpk stays null).
       init: () async {
         _tpk = await initTesterPayKit();
       },
       config: RunAppConfig(
         enableInteraction: kDebugMode, // AI drive — debug only
         monitorHttp: true, // also logs TPK's own API calls in dev
       ),
       child: const MyApp(),
     );
   }
   ```

6. **Wrap the MaterialApp.**
   ```dart
   MaterialApp(
     navigatorKey: _navigatorKey,
     navigatorObservers: [
       MoinsenNavigatorObserver.instance,         // dev route tracking
       if (_tpk != null) _tpk!.navigatorObserver, // TPK screen views — payout-relevant!
     ],
     builder: (context, child) {
       final tpk = _tpk;
       final content = child ?? const SizedBox.shrink();
       if (tpk == null) return content; // TPK off/failed → app runs clean
       return TesterPayKitWrapper(
         sdk: tpk,
         // Required when mounted via builder: the wrapper sits above
         // the app's Navigator and pushes its own screens through this.
         navigatorKey: _navigatorKey,
         child: content,
       );
     },
     home: const HomeScreen(),
   )
   ```

7. **Tester onboarding — how testers identify themselves.**
   - **Default: invite-code in-app.** `enableInviteCodeOnboarding` is
     true by default — when the SDK has no tester identity, the
     wrapper presents an invite-code screen; the tester enters the
     code the coordinator issued them (via the TPK web console /
     recruiting flow) and it persists across launches.
   - **Dev/CI alternative:** bake the code in at build time with
     `--dart-define=TPK_TESTER_CODE=<code>` (handled by the bridge
     above) so automated builds skip the screen.

8. **Verify.**
   ```bash
   flutter pub get
   flutter analyze     # must be clean
   ```
   Then run on a simulator/device:
   ```bash
   flutter run --dart-define=TPK_PROJECT_ID=<uuid>
   ```
   - App boots even with no network (TPK init is non-blocking).
   - Floating feedback button visible; shake the device → bug-report
     dialog appears. Submit a test report, then check it arrived:
   ```bash
   tpk bug list --bundle-id <bundle-id> --limit 5
   ```
   - Navigate through 3–4 screens, end the session → check engaged
     minutes with `tpk session list --limit 5`.

## Pitfalls (learned the hard way)

- **GoRouter / Router 2.0 breaks auto screen-tracking → testers get
  underpaid.** The navigator observer only sees Navigator-1.0 route
  pushes. With GoRouter, call
  `currentTesterPayKit?.trackScreenView('<screen>')` from a GoRouter
  `routerDelegate`/listener on route changes — otherwise engaged
  minutes undercount and payouts come out too low.
- **Integration ≠ payout.** The server pays €0 until the project has
  an active payout rule (`tpk payout-rule list` / bootstrap step 6).
- **Don't treat dev sessions as tester data.** Debug-build taps
  (including AI-driven ones via moinsen) count as activity; keep dev
  noise out of the project you pay real testers on.
- **Web target:** this combo recipe is native-only (`moinsen_runapp`
  needs `dart:io`). For a Flutter Web build, drop `moinsen_runapp` and
  keep only the `testerpaykit` wrapper.

## Optional: semantic tagging & task auto-tracking

Worth doing screen-by-screen once the basic loop works:

- Give important interactive widgets proper `Semantics` labels and
  stable `Key`s (accessibility + makes breadcrumbs/AI testing
  meaningful).
- Wrap widgets that correspond to tester-checklist tasks with TPK's
  `TesterTaskRelevant` marker (zero-overhead `RenderProxyBox`) so the
  SDK auto-detects "task done" instead of relying on manual check-off.
- Wrap PII-bearing widgets with `TesterSensitive` so tester-side
  screenshots mask them.

## What this reference does NOT do

- Doesn't add iOS/Android permission strings (microphone etc.) —
  surface them to the user if a runtime error mentions
  `NSMicrophoneUsageDescription` or similar.
- Doesn't change the host's state-mgmt approach (Riverpod / Bloc) —
  the wrapper is a plain StatefulWidget.
- Doesn't auto-commit. The user reviews the diff and decides.
