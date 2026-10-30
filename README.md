# TesterPayKit

**Crowd testing for apps built with AI.** TesterPayKit gets real, everyday
testers (and AI testers) into your Flutter, Expo or web app with a drop-in
SDK — bug reports, crash capture, sessions, navigation flow and payouts —
and hands the findings straight to your coding agent.

🌐 **Website:** https://testerpaykit.com · 📖 **Docs:** https://testerpaykit.com/docs

> This repository is the **public hub**: the Claude Code plugin, issues and
> discussions. The product's source lives in a private repository; the
> ready-to-use packages are on pub.dev and npm (below).

## Claude Code plugin

Let your coding agent set up and use TesterPayKit:

```bash
claude plugin marketplace add moinsen-dev/TesterPayKit
claude plugin install tpk@testerpaykit
```

You get `/tpk:cli-install`, `/tpk:bootstrap` (creates your project),
`/tpk:integrate` (adds the SDK to a Flutter, Expo or web app), `/tpk:status`
and `/tpk:fix-next-bug`, plus the remote `testerpaykit` MCP server: Claude
orders a test with real people and reads their sessions, verdicts and bug
reports. Connect it once with a token from the console — `/tpk:connect`
walks you through it. Details: [plugins/README.md](plugins/README.md).

## Packages

| Package | Where | Install |
|---|---|---|
| `testerpaykit` — Flutter SDK | [pub.dev](https://pub.dev/packages/testerpaykit) | `flutter pub add testerpaykit` |
| `@testerpaykit/expo` — Expo / React Native SDK | [npm](https://www.npmjs.com/package/@testerpaykit/expo) | `npx expo install @testerpaykit/expo` |
| `tpk_cli` — CLI | [pub.dev](https://pub.dev/packages/tpk_cli) | `dart pub global activate tpk_cli` |

A coordinator account is needed to create projects; sign-up is currently by
invitation (business@moinsen.dev).

## Get help / report something

- 🐞 **Bug?** → open an [Issue](../../issues/new/choose)
- 💡 **Feature idea?** → open an [Issue](../../issues/new/choose)
- 💬 **Question / discussion?** → [Discussions](../../discussions)
- 🔒 **Security issue?** → see [SECURITY.md](SECURITY.md) — please don't open a public issue

## License

The Claude Code plugin in [`plugins/tpk`](plugins/tpk) is MIT-licensed
([LICENSE](plugins/tpk/LICENSE)). The SDKs and the CLI are **commercial
software**, source-available on pub.dev and npm under the TesterPayKit
Commercial License — use as a dependency in your apps is permitted;
redistribution, forking and competing products are not. See each package's
LICENSE.

---
Made by [moinsen-dev](https://moinsen.dev)

<!--
  Generated from the private monorepo (TesterPayKit/public/) by
  `make publish-public` — do not edit this repository directly.
-->
