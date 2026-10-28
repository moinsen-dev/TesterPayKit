# Integrate TesterPayKit — Expo / React Native reference

Platform reference for the `integrate` skill. Read this when Step 0 of
`SKILL.md` detected an Expo / React-Native project (`package.json` has an
`expo` dependency, and/or `app.json` / `app.config.{js,ts}` is present).
The four universal invariants in `SKILL.md` still apply — this file is
the Expo-specific recipe. SDK package: **`@testerpaykit/expo`**.

Goal: an existing Expo/RN app captures tester sessions + bug reports
(shake-to-report, floating feedback button, post-crash) and posts them
to the user's TPK project — without TPK ever blocking or crashing the
host app.

## Prerequisites

- Target is an Expo / React-Native app (`package.json` with `expo`, plus
  `app.json` or `app.config.*`).
- TPK project UUID is known. If unknown, run the `bootstrap` skill first.
- A tester code (the coordinator issues codes via the TPK console /
  recruiting flow). For dev you can use any string, e.g.
  `demo-tester-01` — it resolves to a stable synthetic tester row.
- **Expo Web target?** `@testerpaykit/expo` targets native RN. If the
  app is really a browser app (Expo for Web / plain React), use
  `references/web.md` instead.

## Steps

1. **Verify the project layout.**
   ```bash
   node -e "process.exit(require('./package.json').dependencies?.expo?0:1)" \
     && echo "expo app" || echo "NOT an expo app"
   ls app.json app.config.js app.config.ts 2>/dev/null
   ```

2. **Install the SDK + the native modules you actually use.**
   Native modules are **peer-optional** — without one, that feature
   no-ops (shake → `expo-sensors`, device/app snapshot → `expo-device` +
   `expo-application`, attachments → `expo-file-system`, voice →
   `expo-audio`). Since 0.6.x a dev-mode setup check logs each missing
   optional module + the feature it disables at init, so audit that log
   once after integrating.

   Version note: on SDK **0.6.1+** no uuid polyfill is needed (built-in
   fallback chain). On the older **0.4.1 npm release**,
   `react-native-get-random-values` is REQUIRED or every bug-report
   submit fails silently — install it and import it as the very first
   line of the entry file (see Pitfalls).
   ```bash
   npx expo install \
     @testerpaykit/expo \
     @react-native-async-storage/async-storage \
     expo-sensors expo-device expo-application expo-file-system expo-audio
   ```

   **Then add the config plugin (microphone permission for voice bug
   reports).** Voice records audio; on iOS, touching the mic without an
   `NSMicrophoneUsageDescription` is a **hard native crash** the SDK
   cannot catch. The `@testerpaykit/expo` config plugin (≥0.5.0) injects
   the iOS usage description + Android `RECORD_AUDIO` — add it to the
   `plugins` array in `app.json` / `app.config.*`:
   ```json
   "plugins": [
     ["@testerpaykit/expo", {
       "microphonePermission": "Allow $(PRODUCT_NAME) to record voice bug reports."
     }]
   ]
   ```
   `microphonePermission` is optional (sensible default); `false` opts
   out. On an SDK older than 0.5.0, use expo-audio's own plugin instead:
   `["expo-audio", { "microphonePermission": "…" }]`. **Either way a
   fresh native build is required** — `npx expo prebuild --clean`, a new
   dev-client, or an EAS build. Config-plugin changes never apply via
   Metro fast-refresh, so adding the key without rebuilding won't fix the
   crash.

3. **Decide the build-mode / env gate.**
   Capture only in dev or in explicit beta builds — never in a store
   release without opt-in. Expo exposes build env via `process.env`
   (only `EXPO_PUBLIC_*` vars reach the client bundle) and the global
   `__DEV__`:

   | var | meaning |
   |---|---|
   | `EXPO_PUBLIC_TPK_PROJECT_ID` | project UUID from bootstrap |
   | `EXPO_PUBLIC_TPK_API_URL`    | omit → prod; LAN IP → local dev |
   | `EXPO_PUBLIC_TPK_TESTER_CODE`| dev/CI tester code (skips the in-app screen) |
   | `EXPO_PUBLIC_TPK_ENABLED`    | `"true"` to force-enable in a release build |

   Gate: `__DEV__ || EXPO_PUBLIC_TPK_ENABLED === 'true'`.

4. **Write a fail-safe bridge `src/tpk/tpkClient.ts`.**
   `TpkExpo.init()` does a network round trip (anonymous-JWT exchange +
   server session start) — wrap it so a failure never bubbles into the
   host:
   ```ts
   import { TpkExpo } from '@testerpaykit/expo';

   const ENABLED =
     __DEV__ || process.env.EXPO_PUBLIC_TPK_ENABLED === 'true';
   const PROJECT_ID =
     process.env.EXPO_PUBLIC_TPK_PROJECT_ID ?? '<UUID-from-bootstrap>';
   const API_URL = process.env.EXPO_PUBLIC_TPK_API_URL; // undefined → prod
   const TESTER_CODE =
     process.env.EXPO_PUBLIC_TPK_TESTER_CODE ?? 'demo-tester-01';

   /** Fail-safe init: returns null when gated off or when init throws. */
   export async function initTpk(): Promise<TpkExpo | null> {
     if (!ENABLED) return null;
     if (!PROJECT_ID || PROJECT_ID.startsWith('<')) return null;
     try {
       return await TpkExpo.init({
         projectId: PROJECT_ID,
         testerCode: TESTER_CODE,
         apiBaseUrl: API_URL, // omit for prod
       });
     } catch (e) {
       console.warn('[tpk] init failed — running without TPK', e);
       return null;
     }
   }
   ```
   `init()` caches the anonymous JWT in AsyncStorage (key
   `@testerpaykit/auth`) keyed by `{projectId, testerCode}` and starts a
   real server session, so `submitBug()` lands on a connected timeline.

5. **Mount the SDK — one wrapper (recommended).** `TesterPayKitWrapper`
   is the single-import host integration: wrap your app root and the
   whole TPK surface is wired — bug-red FAB + radial menu, 5-step
   BugReportSheet, GeneralFeedbackDialog, SessionEndDialog, long-press →
   VoiceBugInputSheet, and shake-to-report on native. It is fail-safe by
   construction: when `tpk` is `null` (init gated off, or init threw) the
   FAB still renders and every server-backed action stays disabled — the
   host app runs unchanged.
   ```tsx
   // On SDK 0.4.1 only: import 'react-native-get-random-values' as the
   // FIRST line of the app entry (uuid/Hermes — see Pitfalls). 0.6.1+ needs no polyfill.
   import { TesterPayKitWrapper, type TpkExpo } from '@testerpaykit/expo';
   import { useEffect, useState } from 'react';
   import { initTpk } from './src/tpk/tpkClient';

   const TPK_ENABLED =
     __DEV__ || process.env.EXPO_PUBLIC_TPK_ENABLED === 'true';

   export default function App() {
     const [tpk, setTpk] = useState<TpkExpo | null>(null);
     useEffect(() => { initTpk().then(setTpk); }, []);

     if (!TPK_ENABLED) return <YourRealApp />; // SDK fully absent when gated
     return (
       <TesterPayKitWrapper tpk={tpk}>
         <YourRealApp />
       </TesterPayKitWrapper>
     );
   }
   ```
   Only `children` + `tpk` are needed; everything else (`enableShake`,
   `shakeThresholdG`, `notificationCount`, the AI-tester hooks, …) is
   optional. **Do not hand-wire `FloatingFeedbackButton` yourself** unless
   you have a specific reason — it takes **four required** callbacks
   (`onReportBug` / `onGeneralFeedback` / `onViewDashboard` /
   `onEndSession`) and has **no** `onPress` / `disabled` prop; the wrapper
   supplies all of them for you.

   **Shake-only / no UI shell?** Skip the wrapper and wire `useShake`
   against the `initTpk()` client directly:
   ```tsx
   import { useShake } from '@testerpaykit/expo';
   import { useEffect, useState } from 'react';
   import { Alert } from 'react-native';
   import { initTpk } from './src/tpk/tpkClient';

   const [tpk, setTpk] = useState(null);
   useEffect(() => { initTpk().then(setTpk); }, []);
   useShake({
     enabled: !!tpk,
     onShake: async () => {
       const r = await tpk?.submitBug({
         title: 'Shake-reported bug',
         description: 'User shook the device.',
         currentScreen: 'HomeScreen',
         severity: 'medium',
       });
       if (r) Alert.alert('Bug reported', `Server id: ${r.id}`);
     },
   });
   ```

6. **Track screen views for fair pay.** Engaged-minutes drive payouts.
   With React Navigation, push a breadcrumb on every route change:
   ```tsx
   <NavigationContainer
     onStateChange={(s) => {
       const name = /* current route name from s */;
       client?.trackScreenView(name);
     }}
   >
   ```
   Without this the session still records, but screen-level timeline +
   per-screen engaged minutes undercount.

7. **End the session** when the tester finishes (optional but improves
   fair-pay accounting):
   ```tsx
   await client?.endSession({ reason: 'completed', bugsFound: 1, tasksCompleted: 2 });
   ```

8. **Verify.**
   ```bash
   EXPO_PUBLIC_TPK_PROJECT_ID=<uuid> npx expo start
   ```
   - App boots even with no network (init is non-blocking / fail-safe).
   - Shake the device (or the simulator's "Shake Gesture") → bug-report
     sheet appears, or tap the floating button. Submit a test report,
     then check it arrived:
   ```bash
   tpk bug list --bundle-id <bundle-id> --limit 5
   ```
   - `tpk session list --limit 5` shows the session.

## Pitfalls

- **Missing native module = silent no-op, not a crash.** If shake never
  fires, you forgot `expo-sensors`; if `deviceSnapshot` is empty, you
  forgot `expo-device` / `expo-application`. That's by design — but
  check the install list before assuming a bug.
- **SPA-style navigation undercounts payouts** (see step 6).
- **Integration ≠ payout.** The server pays €0 until the project has an
  active payout rule (`tpk payout-rule list`).
- **Don't count dev sessions as tester data.** Your own `expo start`
  taps count as activity; keep dev noise out of a paid project.
- **Local dev on a real device** needs the Mac's LAN IP, not
  `localhost`: `EXPO_PUBLIC_TPK_API_URL=http://$(ipconfig getifaddr en0):18080`.
  iOS simulator can use `http://localhost:18080`.
- **Redbox `new NativeEventEmitter(null)` / `PushNotificationIOS` at
  startup = stale SDK.** Builds before the static-`AppState`-import fix
  crashed the host on `init()`: a dynamic `import('react-native')` built
  an ESM namespace by enumerating *all* of react-native's exports, which
  fired the deprecated `PushNotificationIOS` getter. This happens too
  early for your env gate, the bridge's `try/catch`, or the provider's
  `onError` to catch — it is only fixable in the SDK. Fixed in
  **`@testerpaykit/expo@>=0.4.1`** (static `AppState` import) — upgrade to
  the latest published version if you hit this.
- **Redbox `crypto.getRandomValues() not supported` = missing polyfill.**
  uuid v9 needs it and Hermes doesn't ship it. Install
  `react-native-get-random-values` and make
  `import 'react-native-get-random-values';` the **first** line of your
  app entry, before any import that touches uuid.
- **Multi-project Metro.** A second Expo project (or `watch_now` etc.)
  can hold `:8081`, so your dev-client remembers the *last* server it saw
  (e.g. `:8082`) and ignores `openurl` port overrides. Fastest fix: serve
  the dev-client on the port it already remembers rather than fighting the
  reload carousel — `npx expo start --port <that-port>`.

## What this reference does NOT do

- The config plugin (step 2) covers the **microphone** permission for
  voice bug reports. Any *other* native permission a host feature needs
  (camera, location, etc.) is still the host's job — surface it if a
  runtime error mentions a missing usage description.
- Doesn't trim/process screen recordings — uploaded as-is.
- Doesn't auto-commit. The user reviews the diff and decides.
