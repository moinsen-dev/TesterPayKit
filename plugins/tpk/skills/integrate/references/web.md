# Integrate TesterPayKit — Web / TypeScript reference

Platform reference for the `integrate` skill. Read this when Step 0 of
`SKILL.md` detected a web app (`package.json` with a web bundler —
`next`, `vite`, or `react-scripts` — and **no** `expo` dependency). The
four universal invariants in `SKILL.md` still apply — this file is the
browser-specific recipe.

SDK packages:
- `@testerpaykit/react` — Context provider + drop-in UI (button, sheet,
  tester hub) + tracking hook.
- `@testerpaykit/sdk` — core `TesterPayKit` (sessions, Outbox, metrics).
- `@testerpaykit/api-client` — `TesterPayKitApiClient` (HTTP + auth,
  including the anonymous-tester JWT exchange).
- `@testerpaykit/storage-browser` — `IndexedDBAdapter` (Dexie/IndexedDB).

Goal: a browser app (React / Next.js / Vite) captures tester sessions +
bug reports and posts them to the user's TPK project — fail-safe, never
blocking render or crashing the host.

## ⚠️ The #1 web-only blocker: CORS / per-project web origins

A browser SDK call is a cross-origin `fetch`. The API rejects origins it
doesn't know, so **before any code**, whitelist the app's origin(s) on
the project — otherwise every TPK call fails in the console with a CORS
error that looks like a TPK bug but isn't:

```bash
tpk project set-origins --id <project-uuid> \
  --origin http://localhost:3000 \
  --origin https://yourapp.com
```

(Repeatable. `tpk project create` also takes `--web-origin` at creation
time; `set-origins` updates an existing project; `--clear` empties it.)
If you see `No 'Access-Control-Allow-Origin'` in the browser console
during verify, this step was skipped or the origin/port doesn't match.

## Prerequisites

- Target is a browser app (`package.json` with `next` / `vite` /
  `react-scripts`, no `expo`).
- TPK project UUID is known. If unknown, run the `bootstrap` skill first.
- The app's dev + prod origins, for the CORS step above.
- A tester code (coordinator issues codes via the console / recruiting).

## Steps

1. **Verify the project layout.**
   ```bash
   node -e "const p=require('./package.json');const d={...p.dependencies,...p.devDependencies};process.exit(d.expo?2:(d.next||d.vite||d['react-scripts'])?0:1)" \
     && echo "web app" || echo "not a plain web app (expo=2 / unknown=1)"
   ```

2. **Whitelist web origins (CORS)** — do the `tpk project set-origins`
   step above first. It's the most common reason a clean integration
   still fails at runtime.

3. **Install.**

   > ⚠️ **Not fully on npm yet.** `@testerpaykit/api-client` + `core` are
   > published, but **`@testerpaykit/react`, `@testerpaykit/sdk`, and
   > `@testerpaykit/storage-browser` are not** — so the line below partly
   > 404s today. The web closure is held back until the web SDK is
   > dogfooded in a real browser app: the Expo SDK shipped a
   > host-crashing bug that only a real-device run surfaced (tests didn't
   > catch it), and we're not repeating that publicly. Until then,
   > integrate from the `moinsen-infrastructure` workspace via path deps.
   > The canonical line below works once the closure is published.
   ```bash
   npm i @testerpaykit/react @testerpaykit/sdk \
         @testerpaykit/api-client @testerpaykit/storage-browser
   ```

4. **Decide the build-mode / env gate.** Capture only in dev or explicit
   beta builds. Only prefixed vars reach the browser bundle —
   `NEXT_PUBLIC_*` (Next.js) or `VITE_*` (Vite):

   | Next.js var | Vite var | meaning |
   |---|---|---|
   | `NEXT_PUBLIC_TPK_PROJECT_ID` | `VITE_TPK_PROJECT_ID` | project UUID |
   | `NEXT_PUBLIC_TPK_API_URL`    | `VITE_TPK_API_URL`    | omit → prod; `http://localhost:18080` → local |
   | `NEXT_PUBLIC_TPK_TESTER_CODE`| `VITE_TPK_TESTER_CODE`| dev/CI tester code |
   | `NEXT_PUBLIC_TPK_ENABLED`    | `VITE_TPK_ENABLED`    | `"true"` to enable in a prod build |

   Gate: `process.env.NODE_ENV !== 'production' || <PREFIX>_TPK_ENABLED === 'true'`.
   The SDK auto-derives `environment` (local vs prod) from the API
   URL's **host** (localhost / 127.* / 10.* / 192.168.* / *.local →
   local) — no separate flag needed.

5. **Browser-only — keep TPK out of SSR.** The SDK and `IndexedDBAdapter`
   touch `window` / `indexedDB`, which don't exist on the server. In
   Next.js App Router: mark the provider file `'use client'`, mount it
   below the server tree, and never import these packages from a Server
   Component. If a bundler still evaluates it server-side, guard with
   `typeof window !== 'undefined'` or `dynamic(() => import('./Tpk'), { ssr: false })`.

6. **Acquire a tester identity (anonymous JWT).** Trade `{projectId,
   code}` for a JWT; the client stores the token and returns the
   resolved `testerId`. Fail-safe wrap:
   ```ts
   import { TesterPayKitApiClient } from '@testerpaykit/api-client';

   export async function authTester(baseURL: string | undefined, projectId: string, code: string) {
     const api = new TesterPayKitApiClient({ apiKey: '', baseURL });
     try {
       const { testerId } = await api.loginAnonymous(projectId, code);
       return { api, testerId: testerId ?? code };
     } catch (e) {
       console.warn('[tpk] tester auth failed — running without TPK', e);
       return null; // host renders clean
     }
   }
   ```

7. **Mount the SDK. Two ways — pick by who your testers are.**

   **(a) Real external testers (anonymous-JWT path, recommended).**
   Build the pre-authed client + IndexedDB storage and drive the core
   SDK directly. This mirrors the Expo/Flutter tester-code model:
   ```tsx
   'use client';
   import { TesterPayKit } from '@testerpaykit/sdk';
   import { IndexedDBAdapter } from '@testerpaykit/storage-browser';
   import { useEffect, useState } from 'react';
   import { authTester } from './tpkAuth';

   const ENABLED =
     process.env.NODE_ENV !== 'production' ||
     process.env.NEXT_PUBLIC_TPK_ENABLED === 'true';

   export function useTpk() {
     const [sdk, setSdk] = useState<TesterPayKit | null>(null);
     useEffect(() => {
       if (!ENABLED || typeof window === 'undefined') return;
       let live = true;
       (async () => {
         const auth = await authTester(
           process.env.NEXT_PUBLIC_TPK_API_URL,
           process.env.NEXT_PUBLIC_TPK_PROJECT_ID!,
           process.env.NEXT_PUBLIC_TPK_TESTER_CODE ?? 'demo-tester-01',
         );
         if (!auth || !live) return;
         const instance = new TesterPayKit({
           apiKey: '',                     // auth rides on the pre-authed apiClient
           projectId: process.env.NEXT_PUBLIC_TPK_PROJECT_ID!,
           testerId: auth.testerId,
           storage: new IndexedDBAdapter(),
           apiClient: auth.api,            // already holds the anonymous JWT
           deviceMetadata: { os: 'Web', osVersion: navigator.userAgent, model: 'browser', manufacturer: 'web', isPhysicalDevice: true },
           appMetadata: { bundleId: 'com.example.web', version: '1.0.0', buildNumber: '1' },
           autoStartSession: true,
         });
         await instance.init();            // returns fast; uploads via Outbox
         if (live) setSdk(instance); else instance.endSession();
       })();
       return () => { live = false; sdk?.endSession(); };
     }, []);
     return sdk; // null until ready → call sites no-op via `?.`
   }
   ```
   Then `sdk?.trackScreenView(...)`, `sdk?.submitBug({ title, description, severity })`.

   **(b) Coordinator/dev embedding (apiKey path, fastest UI).** When the
   surface is your own console/dashboard and you have a coordinator API
   key, `@testerpaykit/react`'s provider wires storage + client + UI for
   you:
   ```tsx
   'use client';
   import { TesterPayKitProvider, TpkBugReportButton, TpkBugReportSheet } from '@testerpaykit/react';

   <TesterPayKitProvider
     apiKey={process.env.NEXT_PUBLIC_TPK_API_KEY!}
     projectId={process.env.NEXT_PUBLIC_TPK_PROJECT_ID!}
     testerId="dev-tester"
     deviceId="web-dev"
     baseURL={process.env.NEXT_PUBLIC_TPK_API_URL}
     deviceMetadata={{ os: 'Web', osVersion: navigator.userAgent, model: 'browser', manufacturer: 'web', isPhysicalDevice: true }}
     appMetadata={{ bundleId: 'com.example.web', version: '1.0.0', buildNumber: '1' }}
   >
     <YourApp />
   </TesterPayKitProvider>
   ```
   The provider exposes `useTesterPayKit()` →
   `{ trackScreenView, trackTap, trackEvent, trackError, sdk }`, plus
   drop-in UI: `TpkBugReportButton`, `TpkBugReportSheet`,
   `TpkTesterLauncher`, `TpkTesterHub`. Don't ship a coordinator apiKey
   in a build you hand to outside testers — use path (a) for those.

8. **Track route changes for fair pay.** SPA navigation doesn't fire a
   page load, so engaged-minutes undercount unless you call
   `trackScreenView` on every route change — Next.js: a `usePathname()`
   effect; React Router: a `useLocation()` effect; Vue Router: an
   `afterEach` hook.

9. **Verify.**
   ```bash
   npm run dev   # open the app
   ```
   - App renders even with the API down (init is non-blocking / wrapped).
   - Trigger a bug report (the button, or call `sdk.submitBug(...)`),
     then confirm it landed:
   ```bash
   tpk bug list --bundle-id <bundle-id> --limit 5
   ```
   - **Watch the browser console + Network tab.** A CORS error =
     step 2 (`set-origins`) is missing or the origin/port mismatches.
     A `window is not defined` build error = step 5 (SSR leak).

## Pitfalls

- **CORS** — re-read the top of this file. Single most common failure.
- **SSR leak** — importing `@testerpaykit/sdk` / `storage-browser` from
  a Server Component or at module top-level breaks the build.
- **IndexedDB unavailable** (Safari private mode, some embedded
  webviews) — auth + Outbox persistence degrade; init still resolves,
  but expect reduced offline durability. Don't block render on it.
- **SPA route tracking undercounts payouts** (see step 8).
- **Integration ≠ payout.** €0 until the project has an active payout
  rule (`tpk payout-rule list`).
- **Don't count dev sessions as tester data** — keep your own
  `npm run dev` activity out of a paid project.

## What this reference does NOT do

- Doesn't build a tester login UI beyond the anonymous-code exchange —
  for email/magic-link, `api.requestMagicLink({ email, returnUrl, country })`
  exists but the UI is the app's job.
- Doesn't configure your bundler / framework — only the TPK wiring.
- Doesn't auto-commit. The user reviews the diff and decides.
