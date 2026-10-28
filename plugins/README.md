# TesterPayKit Claude Code plugins

The official Claude Code plugin marketplace for
[TesterPayKit](https://testerpaykit.com) — the crowd-testing platform
for AI-built apps. The marketplace manifest sits at the repository root
(`.claude-plugin/marketplace.json`); the plugins live here.

## Current plugins

| Plugin | Description |
|---|---|
| **tpk** | Drives the TesterPayKit crowd-testing lifecycle from inside a Claude Code session — install the `tpk` CLI from pub.dev, bootstrap a project (incl. payout rule), integrate an SDK, query sessions/bugs/recruiting, and close the agentic feedback loop with `/tpk:fix-next-bug`. All skills are namespaced `/tpk:*`. Also registers the read-only `tpk` MCP server (`tpk_mcp`, ships with the CLI). |

## Supported SDK targets (`/tpk:integrate`)

| Target | Package | Status |
|---|---|---|
| Flutter (iOS/Android/desktop/web) | `testerpaykit` (pub.dev), default combo with `moinsen_runapp` | Published, reference SDK |
| Expo / React Native | `@testerpaykit/expo` (npm) | Published |
| Web / TypeScript (React, Next.js, Vite) | `@testerpaykit/react` + `@testerpaykit/sdk` | Built, not yet on npm (early access) |
| Native Swift (iOS 15+ / macOS 12+) | `TesterPayKit` Swift Package — full behavioral parity + built-in SwiftUI tester experience (floating feedback button, bug report, voice input, screenshot annotation, tester dashboard, session-end incl. demand-signal questions) | Early access (path dependency; contact business@moinsen.dev) |

The Flutter, TypeScript and Swift clients speak the same wire contract
(`sdk-contract.yaml`, validated 28/28 on each): anonymous tester auth,
durable offline outbox, crash capture, weekly earnings submission with
a server-verified integrity hash, and the phase-3 demand signal
("would you use / pay for / recommend it?").

## Install

```bash
claude plugin marketplace add moinsen-dev/TesterPayKit
claude plugin install tpk@testerpaykit
```

You get five skills (`/tpk:cli-install`, `/tpk:bootstrap`,
`/tpk:integrate`, `/tpk:status`, `/tpk:fix-next-bug`) plus the `tpk` MCP server.
The CLI itself installs with `dart pub global activate tpk_cli` —
the `cli-install` skill walks Claude through it.

## Layout

```
.claude-plugin/marketplace.json   # registry — names + paths of plugins
plugins/
└── tpk/
    ├── .claude-plugin/plugin.json  # manifest + MCP server
    ├── LICENSE                     # MIT
    └── skills/                     # bootstrap, integrate (+ references/),
                                    # status, cli-install, fix-next-bug
```

## Updating

```bash
claude plugin marketplace update testerpaykit
```

## License

MIT — see [tpk/LICENSE](./tpk/LICENSE).
