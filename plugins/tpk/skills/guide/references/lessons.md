# TesterPayKit guide — lessons

Nine lessons. Each has the same parts: **Goal · Predict (optional) ·
Explain · Picture · Try · Check · Take-away**. Teach them in that order
(see `SKILL.md`). Content checked against the platform on 2026-11-06;
amounts are **pilot starting values** and may change.

## Course map and order by goal

| # | Lesson | ~min |
|---|---|---|
| 1 | The idea: three sides, one loop | 4 |
| 2 | The building blocks: console, CLI, MCP, SDK | 5 |
| 3 | Connecting: account, token, project | 5 |
| 4 | The SDK in your app: what a report carries | 6 |
| 5 | Who tests: person, phone number, approval | 5 |
| 6 | Test orders and checklists | 6 |
| 7 | The life of a bug report | 6 |
| 8 | Money: what testers earn and what you pay | 6 |
| 9 | The agent loop: Claude fixes, testers verify | 6 |

Order by the learner's first goal (then the remaining lessons in number
order):

| Goal | Start with |
|---|---|
| understand the idea | 1, 2, 7, 8 |
| get real testers on my app | 1, 2, 3, 4, 5, 6 |
| let Claude fix tester bugs | 1, 2, 3, 7, 9 |
| understand payment | 1, 8, 5, 6 |

UI labels the tester sees (German / English): **Bug melden** / Report Bug ·
**Feedback** · **Meine Übersicht** / My Workboard · **Meine Bugs** / My Bugs
· **Meine Nachtests** / My Retests · **Mein Geld** / My money ·
**Nachrichten** / Messages.

---

## Lesson 1 — The idea: three sides, one loop

**Goal.** Explain in two sentences what TesterPayKit does and for whom.

**Predict.** "Who has to be happy for a test platform like this to work —
only you as the app owner?" (Open answer; then reveal: three sides.)

**Explain.**
- Real people (students, retail and gastro workers, parents, retirees) use
  your app the way your users will, and report what goes wrong — by voice
  or typing, with a screenshot and context attached automatically.
- You (or Claude) fix it; the person who found it, or another tester,
  checks the fix in the new version.
- Testers are paid fairly for honest work. The principle: it only works if
  **all three sides say yes** — testers ("effort and pay are fair"),
  customers ("the result is worth the money") and the platform ("building
  and running it pays off"). That is why amounts are learned in the pilot,
  not fixed forever.
- Focus: German-speaking market (DE/AT/CH), apps built fast with AI tools.

**Picture.**
```
  Tester ──report──▶ TesterPayKit ──bug + context──▶ You / Claude
     ▲                                                  │
     └──────── retest in the new version ◀── fix ───────┘
```

**Try.** Nothing to run. Ask the learner to name their own app and what
they would most like real people to find in it. Save the app name in
`setup.project` if there is none yet.

**Check.**
- 1a. *Why are payment amounts in the pilot not fixed?*
  - ✔ They are learned so that testers, customers and the platform all
    find them fair.
  - ✘ Because testers negotiate their own rates. — No negotiation; the
    platform sets rates.
  - ✘ Because payment only starts after launch. — Bookings happen now (as
    play money, lesson 8).
- 1b. *What does a tester send besides their words?*
  - ✔ Screenshot, current screen, recent steps, device and app version.
  - ✘ Only the text they typed. — Context is attached automatically.
  - ✘ A screen recording of the whole session. — Not by default.

**Take-away.** Real people find, Claude fixes, people confirm — and it
only lasts if all three sides win.

---

## Lesson 2 — The building blocks

**Goal.** Know which tool to reach for: console, CLI, MCP server or SDK.

**Explain.**
- **Console** — `app.testerpaykit.com`. The web app for people: projects,
  testers, bug reports, waitlist, money, access tokens.
- **SDK** — a small package inside your app (Flutter, Expo/React Native,
  Web/React; native Swift in early access). It shows testers the bug
  button and sends reports. All four SDKs ship together with the same
  version and features.
- **CLI `tpk`** — for terminals and agents: log in, create projects, list
  sessions and bugs, and *change* bug status (`tpk bug fix`).
- **MCP server `testerpaykit`** — lets Claude order tests with real
  people, read results and bugs, keep a checklist, and look at your own
  running app. Reading bugs works here; changing bug status does not (that
  is the CLI or console, on purpose).
- **Skills** in this plugin wrap all of that: `/tpk:connect`,
  `/tpk:bootstrap`, `/tpk:integrate`, `/tpk:status`, `/tpk:live`,
  `/tpk:fix-next-bug`.

**Picture.**
```
 your app + SDK ──▶ api.testerpaykit.com ◀── console (browser)
                          ▲        ▲
                     tpk CLI    MCP server ◀── Claude Code
```

**Try.** Show the setup detected at the start as a small table: CLI
installed / logged in / MCP connected / SDK in this repo. For each "no",
name the skill that fixes it — do not run it.

**Check.**
- 2a. *Claude should mark a bug as fixed. Which path does that?*
  - ✔ The CLI (`tpk bug fix`) — or a person in the console.
  - ✘ The MCP server. — It only reads bugs; status changes are kept off it
    deliberately.
  - ✘ The SDK. — That is the tester's side inside the app.
- 2b. *Where does a tester tap "Bug melden"?*
  - ✔ Inside your app, through the SDK.
  - ✘ In the console. — The console is for you, not for testers in the app.
  - ✘ In a separate TesterPayKit app. — Not yet; testers work inside the
    app they test.

**Take-away.** Testers live in the SDK, you live in the console, Claude
lives in CLI + MCP.

---

## Lesson 3 — Connecting: account, token, project

**Goal.** Have a working connection, or know exactly which step is missing.

**Explain.**
- **Account** — sign in at `app.testerpaykit.com`.
- **CLI login** — `tpk login` opens the browser once (device login).
- **Token for Claude** — console → Profile → *Access tokens for Claude
  Code*. It starts with `tpk_pat_`, is shown once, and goes into the shell
  profile as `TPK_API_TOKEN` — never into the chat, never into a command
  line that gets saved.
- **Project** — one per app, identified by its bundle id (iOS/Android) or
  web address. `/tpk:bootstrap` creates it and prints the values the app
  needs.

**Try.** Run read-only: `tpk --json whoami`, `tpk --json project list`,
and `list_test_orders` if the MCP tools are there. Show what works. For
the first missing piece offer the matching skill (`/tpk:cli-install`,
`/tpk:connect`, `/tpk:bootstrap`). Update `setup`.

**Check.**
- 3a. *Where does the access token belong?*
  - ✔ In the environment variable `TPK_API_TOKEN` in the shell profile.
  - ✘ Pasted into the chat so Claude can use it. — Never; chats are
    stored.
  - ✘ In the `claude mcp add --header` command. — That saves it in plain
    text in the Claude config.
- 3b. *What tells TesterPayKit which app a report belongs to?*
  - ✔ The project with its bundle id or web address.
  - ✘ The tester's phone number. — That identifies the person, not the
    app.
  - ✘ The SDK version. — Same version for all apps.

**Take-away.** Account, CLI login, token in the environment, one project
per app — that is the whole wiring.

---

## Lesson 4 — The SDK in your app

**Goal.** Know what the SDK does in your app and what a report carries.

**Predict.** "The TesterPayKit server is down while a tester sends a
report. What should happen to your app?" (Then reveal: nothing — the app
keeps running, the report waits on the device.)

**Explain.**
- The SDK wraps your app. Testers see a small **bug button** (or shake the
  phone). Regular users never see it — it appears only for confirmed,
  approved testers.
- A report carries: the screenshot (taken before any dialog covers it),
  the current screen, the last 20 steps (taps, screen changes), device
  model and app version.
- **Voice**: the tester speaks, the platform turns it into a structured
  report (title, what happened, what was expected) in the tester's
  language; they check it before sending. Your team also gets an English
  version.
- **Fail-safe**: nothing in the SDK may block or crash your app. Offline,
  the report is stored on the device and sent later; each report has a key
  so it never arrives twice.
- Also in the tester menu: **Meine Bugs**, **Meine Nachtests**, **Mein
  Geld**, **Nachrichten**, a daily briefing on the first tap of the day.

**Picture.**
```
 tap bug / shake ─▶ screenshot + screen + 20 steps + device
                 ─▶ speak or type ─▶ check ─▶ send
                                     offline? ─▶ queue on device ─▶ later
```

**Try.** Look in the current repo for the SDK (detected at the start). If
it is there, show the line where the app is wrapped and name the platform.
If not, say `/tpk:integrate` does it and which SDK fits their stack.
Optional, on a yes and only if they are a confirmed tester themselves:
send yourself a test report from the app and find it with `list_bugs`.

**Check.**
- 4a. *Who sees the bug button?*
  - ✔ Only confirmed and approved testers.
  - ✘ Every user of the app. — Regular users never see it.
  - ✘ Only you as the developer. — Testers need it too.
- 4b. *A tester is offline and sends a report. What happens?*
  - ✔ It is stored on the device and sent once they are online.
  - ✘ The report is lost. — The queue keeps it.
  - ✘ The app shows an error and blocks until online. — The SDK never
    blocks the app.

**Take-away.** The SDK adds a quiet tester layer with full context, and it
never gets in your app's way.

---

## Lesson 5 — Who tests: person, phone number, approval

**Goal.** Explain how a stranger becomes a tester of your app, and why
the phone number matters.

**Explain.**
- **A person = a confirmed phone number** (SMS, DE/AT/CH numbers). One
  person can use several phones and several apps; all of them belong to
  the same person, and so does their money.
- **Approval** — the TesterPayKit team approves people before they can
  report. That plus the confirmed number is the protection against fake
  accounts. Devices are only an early-warning signal (many devices at
  once raise an alert; a device can be blocked as an emergency brake).
- **Handle** — every tester gets a readable name like `brave-salmon`, so
  reports are never anonymous to you, without exposing personal data.
- **Getting testers into your app** — a public **waitlist** page for your
  project; you pick people and invite them. For iOS betas the invite can
  go straight into TestFlight (Apple sends the mail). In the app the
  person taps something like "Beta-Tester werden", confirms their number,
  and once approved the bug button appears.
- *Not yet:* waitlist actions through Claude (MCP) — today in the console.

**Picture.**
```
 waitlist sign-up ─▶ confirmed email ─▶ you invite (TestFlight)
   ─▶ installs app ─▶ "Beta-Tester werden" ─▶ SMS code
   ─▶ approved ─▶ bug button appears
```

**Try.** If the MCP tools are connected, `list_bugs` for their project and
show the reporter handles (handles only, nothing else personal). Otherwise
explain where the waitlist lives in the console: the project's *Warteliste*
page.

**Check.**
- 5a. *A tester uses your app on an iPhone and an Android tablet. How many
  start bonuses do they get?*
  - ✔ One — money belongs to the person, not the device.
  - ✘ Two, one per device. — The person is the number, not the device.
  - ✘ None, two devices are not allowed. — More devices are fine.
- 5b. *What actually protects against fake testers?*
  - ✔ The SMS-confirmed phone number plus approval by the team.
  - ✘ Allowing only one device per number. — There is no device id that
    is the same across apps of different vendors, so that rule was
    dropped.
  - ✘ The email address. — Easy to create many; not the protection.

**Take-away.** One confirmed number is one person, the team approves
people, and the waitlist plus TestFlight bring them to your app.

---

## Lesson 6 — Test orders and checklists

**Goal.** Know how to ask real people for a test or for opinions — and
where money starts.

**Explain.**
- **Checklist** — what testers should test in your app: one task per flow,
  3–8 concrete steps that each end in something visible. Testers tick
  steps off in the app; you read the verdicts
  (`get_checklist`, `set_checklist_task`, `get_checklist_progress`).
- **Test order** — a listing for real people: app, platforms, number of
  testers, reward per tester, tasks. **Creating is free** and keeps it
  closed. **Publishing costs money**: the first call returns the price
  (reward total + 25 % service fee + VAT) and a confirm token; only a
  second call with that token acts, and you pay yourself in the browser.
- **Feedback order** — opinions instead of tests: people look at a URL or
  description and answer fixed questions (what is it, would you use / pay
  / recommend it) plus up to 8 of your own. It has an invite link for your
  own people.
- Results are marked `origin: "human"` or `origin: "ai"` (TesterPayKit's
  AI testers); by default you only get human results.

**Picture.**
```
 create (free, closed) ─▶ publish: price + token ─▶ YOUR yes
   ─▶ pay in browser ─▶ open ─▶ testers work ─▶ results ─▶ accept = pay
```

**Try.** Read-only: `list_test_orders`; if the project has a checklist,
`get_checklist`. Optional, on a yes: create a small *draft* order for
their app and discard it right away with `discard_test_order` — to see the
shape. Never publish in the guide.

**Check.**
- 6a. *Claude created a test order. What has it cost so far?*
  - ✔ Nothing — creating is free and the listing stays closed.
  - ✘ The full reward total. — Only publishing costs.
  - ✘ The 25 % service fee. — Also only on publishing.
- 6b. *What does a good checklist step end with?*
  - ✔ Something the tester can see on screen.
  - ✘ A question about how they feel. — That is a feedback order.
  - ✘ A link to the source code. — Testers do not read code.

**Take-away.** Drafts are free, publishing needs your explicit yes and
your own payment, and good checklist steps end in something visible.

---

## Lesson 7 — The life of a bug report

**Goal.** Follow one report from "sent" to "confirmed fixed" and know what
the tester hears at each step.

**Predict.** "A tester sends a report and hears nothing for days. What do
you think they do next time?" (Then: they stop reporting — that is why
TesterPayKit tells them when someone looks.)

**Explain.**
- **submitted** — arrives; the tester gets a thank-you (no amount).
- **First look** — when someone in your project opens it (console,
  `tpk bug context`, MCP `get_bug`), the tester hears once: "Deine Meldung
  wird angeschaut". Status does not change.
- Then you decide: **accepted** (real bug), **duplicate** (already
  reported by someone else), **rejected** (not a bug / not usable).
  `underReview` exists for "we are on it".
- **fixed** — you ship a fix with a version. A named retester (or the AI
  tester, if switched on) checks it in that version → **verified**, or it
  goes back to **submitted** with the failing step noted.
- Every change the tester cares about lands in **Nachrichten**, as a badge
  on the bug button and in the daily briefing. In **Meine Bugs** they see
  every report with its status.

**Picture.**
```
 submitted ─(first look: "wird angeschaut")─▶ accepted ─▶ fixed ─▶ verified
     │                                                     │
     ├─▶ duplicate                      retest fails ◀─────┘ back to submitted
     └─▶ rejected
```

**Try.** If MCP is connected: `list_bugs` for their project, pick the
newest, `get_bug` it, and walk through the fields (steps, screen, device,
original words vs English version). Say that opening it just sent that
tester the "wird angeschaut" note — that is the point.

**Check.**
- 7a. *You open a new report in the console and close it again. What
  changed for the tester?*
  - ✔ They get one note that their report is being looked at.
  - ✘ Nothing — only a status change notifies. — The first look notifies.
  - ✘ The report becomes "accepted". — Status stays "submitted".
- 7b. *The retest fails. Where does the bug go?*
  - ✔ Back to "submitted", with the failing step in the notes.
  - ✘ To "rejected". — The bug is real; the fix was not.
  - ✘ It stays "fixed". — A failed retest reopens it.

**Take-away.** Every report gets an answer, and a fix only counts once a
person has seen it work.

---

## Lesson 8 — Money

**Goal.** Explain what testers earn for what, how long it takes, and what
you pay. All amounts are **pilot starting values**.

**Explain.**
- **Per bug report**: a small base amount (0.20 €; at most 10 a day per
  person; not for general feedback; held if the report is rejected).
- **Bug bonus** when your team accepts or fixes it, by severity: minor
  2 €, medium 4 €, severe or crash 8 €. The tester is told only "bugs are
  rewarded by severity and whether they are new" — no amount right after
  sending, so it never feels like "money per click".
- **Duplicate** of someone else's bug: 0.50 € for independently confirming
  it. **Retest**: 3 € per checked fix, at least 5 € per round.
  **Idea bonus**: you can give one to a report.
- **Time**: only for orders with an announced duration — 0.30 € per
  announced minute (18 €/h), at least 5 € per order. Faster is not
  punished.
- **Start bonus** 3 € once per person, held until their first approved
  work.
- **Review period**: what the tester triggers waits 3 working days (you
  can release earlier or hold, with a note they see); what your team
  decides is released at once.
- **Play money today**: in the pilot all bookings are real entries but no
  money moves; every project has a play budget and everything is labelled
  as play money. Later, real payouts run through Stripe from a deposit you
  pay first.
- **What you pay** for a published test order: reward total + 25 % service
  fee + VAT.

**Picture.**
```
 report ─▶ 0.20 € in review (3 working days) ─▶ released
 accepted/fixed ─▶ bug bonus 2/4/8 € ─▶ released at once
 rejected ─▶ base amount held
```

**Try.** No live money calls. Ask the learner to compute one example with
you: "A tester sends 3 reports today; you accept one as medium, mark one
as a duplicate of another person's bug, reject one. What is booked?"
(Answer: 3 × 0.20 € base, 4 € bonus, 0.50 € duplicate, and the rejected
report's 0.20 € is held → 4.90 € released over time, 0.20 € held.)

**Check.**
- 8a. *Why does the thank-you after sending show no amount?*
  - ✔ So reporting does not feel like money per click — quality and
    novelty count.
  - ✘ Because the amount is not known yet. — The base amount is known.
  - ✘ Because testers are paid only at the end of the month. — Bookings
    are visible in "Mein Geld" right away.
- 8b. *A tester finishes a 30-minute announced task in 20 minutes. What
  is booked?*
  - ✔ 30 minutes — the announced time counts, faster is not punished.
  - ✘ 20 minutes. — Measured time counts only for honest partial work.
  - ✘ Nothing until a coordinator approves. — It is booked and released
    after the review period.

**Take-away.** Small base, real reward for real bugs, announced time paid
in full — and today it is play money, clearly labelled.

---

## Lesson 9 — The agent loop

**Goal.** Run the loop "tester reports → Claude fixes → tester verifies"
and know where your yes is needed.

**Explain.**
- `/tpk:fix-next-bug` takes the most important open bug, loads its full
  context (`tpk bug context`), proposes and builds the fix, and marks it
  fixed with the version (`tpk bug fix <id> --version …`). Name a retester
  with `--tester` so a person checks it; otherwise the AI tester (if
  switched on) checks, or your word counts.
- `/tpk:live` connects Claude to your own app on your own phone:
  screenshot, screen texts, notes you send from the device, and tapping
  through a checklist. Only your device, only when you switch it on.
- **Where your yes is needed**: anything that costs money (publishing,
  accepting results) and every deploy of your app. Marking a bug fixed can
  release money too — review what Claude marks.

**Picture.**
```
 /tpk:fix-next-bug: pick ─▶ context ─▶ fix ─▶ tpk bug fix --version --tester
                                                     │
 tester: "Meine Nachtests" ─▶ steps in new version ─▶ verified / reopened
```

**Try.** Read-only: `tpk --json bug list --status submitted --limit 3`
(or `list_bugs`). If there is an open bug, show which one
`/tpk:fix-next-bug` would take and why. Do not run the fix inside the
guide — offer it as the learner's next step.

**Check.**
- 9a. *Claude marks a bug fixed without naming a retester and the AI
  tester is off. What happens?*
  - ✔ The bug counts as confirmed at once and the reward is paid — so
    review before marking.
  - ✘ It waits forever for a retest. — Without anyone to check, your word
    counts.
  - ✘ The tester who reported it gets a retest automatically. — Only a
    named retester gets one.
- 9b. *Which of these may Claude do without asking you?*
  - ✔ Read a bug and propose a fix.
  - ✘ Publish a test order. — Costs money, needs your yes.
  - ✘ Accept a tester's result. — Pays money, needs your yes.

**Take-away.** Claude does the work between report and retest; money and
releases stay your decision.
