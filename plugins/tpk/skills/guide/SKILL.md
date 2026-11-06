---
name: guide
description: >
  Interactive getting-started course for TesterPayKit: teaches a new
  developer or app owner how the system works, one short lesson at a time,
  with small hands-on steps in their own setup, check questions and saved
  progress they can resume later. Trigger when the user says "tpk guide",
  "explain TesterPayKit", "how does TPK work", "I'm new to TesterPayKit",
  "onboard me", "teach me TPK", "Einstieg", "erklär mir TesterPayKit",
  "TPK-Kurs", "weiter mit dem Kurs", "wo war ich im Guide", or asks for a
  tour of the platform. Read-only by default: it never spends money and
  never changes TPK state without an explicit yes.
---

# TesterPayKit guide

A course, not a manual. The learner should leave each lesson able to *do*
something and explain *why* it works that way. Lesson content lives in
[`references/lessons.md`](references/lessons.md); read only the lesson
you are about to teach, not the whole file.

## Arguments

| Call | Does |
|---|---|
| `/tpk:guide` | resume where the learner stopped (or start) |
| `/tpk:guide status` | show the course map with progress, then stop |
| `/tpk:guide <n>` or `lesson <n>` | jump to lesson n (mark skipped ones as `skipped`, not `done`) |
| `/tpk:guide review` | only the review questions that are due |
| `/tpk:guide reset` | ask once, then delete the progress file |

## Progress file

Path: `${XDG_CONFIG_HOME:-$HOME/.config}/testerpaykit/guide-progress.json`.
Per person, not per repository. Read it with your file tools at the start
of every call; create the folder when it is missing. Write it after every
lesson step that changed something (not only at the end), so an
interrupted session loses nothing.

```json
{
  "version": 1,
  "language": "de",
  "role": "developer",
  "goal": "first real testers on my Flutter app",
  "setup": {"cli": true, "logged_in": true, "mcp": false, "sdk": "flutter", "project": "Kinpal"},
  "current": 4,
  "lessons": {
    "1": {"status": "done", "completed_at": "2026-11-06", "check": "2/2", "tried": true},
    "4": {"status": "in_progress", "step": "try"}
  },
  "review": [{"lesson": 2, "q": "2b", "due_after_lesson": 4}],
  "updated_at": "2026-11-06T18:20:00Z"
}
```

- `status`: `done`, `in_progress`, `skipped`.
- `step`: `explain`, `try`, `check` — where to pick up inside a lesson.
- `review`: questions answered wrong. Ask each again once its
  `due_after_lesson` is done; remove it when answered right.
- `pending_check`: set it **before** you show a check question, e.g.
  `{"q": "1a", "options": {"A": "wrong-negotiate", "B": "right", "C":
  "wrong-after-launch"}}` with the shuffled order. An answer that arrives
  in a later session ("A") can then still be graded. Clear it once graded.
- `updated_at`: real time from `date -u +%Y-%m-%dT%H:%M:%SZ`, never made up.
- Store nothing secret: no tokens, no phone numbers, no email addresses.

**Saving.** Write the whole file with the Write tool. The file lives
outside the repository, so Claude Code may ask the learner once for
permission. Say "saved" only when the write succeeded. If it was refused,
say so and show the JSON so nothing is lost.

## Flow

### First call (no progress file)

1. Greet in one sentence and say what the course is: 9 short lessons, about
   5 minutes each, progress is saved, stop any time.
2. **Detect the setup yourself** before asking anything — all read-only:
   - `tpk --version` and `tpk --json whoami` (CLI installed? logged in?)
   - `TPK_API_TOKEN` set? Check with `test -n "$TPK_API_TOKEN" && echo set`
     — never print the value.
   - Are the `testerpaykit` MCP tools available in this session?
   - Is an SDK in the current repo? Look for `testerpaykit` in
     `pubspec.yaml`, `@testerpaykit/` in `package.json`, `TesterPayKit` in
     `Package.swift` / the Xcode project.
3. Ask **one** question with AskUserQuestion: what the learner wants first
   (e.g. "understand the idea", "get real testers on my app", "let Claude
   fix tester bugs", "understand payment"). Use it to pick the order —
   the map in `lessons.md` says which lessons each goal needs first.
4. Write the progress file, then start the first lesson.

### Resume

1. Read the progress file. Show a three-line recap: where they stopped,
   what is next, how many lessons are done.
2. If a review question is due, ask it first (retrieval before new
   material). One question, then move on.
3. Continue at `current` / `step`.

### One lesson

Every lesson in `lessons.md` has the same parts. Teach them in order and
stop for the learner where marked:

1. **Goal** — one sentence: what they can do after this lesson.
2. **Predict** (if the lesson has one) — ask the prediction question
   *before* explaining. Wrong guesses are fine; say so. It makes the
   explanation stick.
3. **Explain** — the key ideas in your own words, at most ~150 words,
   plus the picture from the lesson (plain-text diagrams render in every
   terminal). Connect it to their setup and goal ("in your Kinpal app this
   means …").
4. **Try** — the hands-on step in their own environment. Read-only steps
   you run yourself and show the result. Steps that change anything
   (creating a draft order, sending themselves a bug report) are optional:
   offer them, do them only on a yes. If their setup is missing a piece,
   say what is missing, offer the skill that fixes it (`/tpk:connect`,
   `/tpk:cli-install`, `/tpk:bootstrap`, `/tpk:integrate`) and mark
   `tried: false` — never block the lesson on it.
5. **Check** — the check questions via AskUserQuestion (2–4 options,
   shuffled, no "(Recommended)" label). After the answer, say right or
   wrong in one line and explain *why* using the lesson's notes on each
   wrong option. Wrong answers go into `review`.
6. **Take-away** — the lesson's one-sentence take-away, then the progress
   bar (one ■ per lesson with status `done`, e.g. `■■■■□□□□□ 4/9`) and the
   next lesson's title. Save. Ask whether to
   go on; on "weiter"/"next" continue, otherwise stop.

### End of the course

Show the map with everything ticked, a five-line summary of what they now
know, the review questions still open, and their next real step based on
their `setup` (e.g. "run `/tpk:integrate`" or "order your first test with
`create_test_order`").

## Didactic rules

- **Short and concrete.** One idea at a time. Use the learner's own app
  and project names once known. No wall of text; if an explanation runs
  long, split it with a question.
- **Their language.** Teach in the language the learner writes in and
  save it as `language`. Name UI labels as the tester sees them (German
  and English labels are both in `lessons.md`).
- **Honest about the state of the product.** Amounts are pilot starting
  values and labelled as such; features marked *early access* or *not yet*
  in `lessons.md` are said that way. Never invent numbers, testers or
  results.
- **Let them answer.** Never answer your own check question in the same
  message. Never reveal the answer in the option labels.
- **Adapt.** If they answer both checks of a lesson right at once, keep
  the next one brisk. If they get one wrong twice, explain it a different
  way (example instead of rule) before moving on.
- **Questions from the learner win.** Answer them, then return to the
  lesson step you were at.

## Safety

- **Money is the human's decision.** In this guide never call
  `publish_test_order`, `accept_result`, `reject_feedback`, never open a
  `checkout_url`. If a lesson shows how they work, show the shape, not a
  live call.
- Optional write steps (a draft order, a self-sent report) need an
  explicit yes and must be cleaned up in the same lesson
  (`discard_test_order` for a draft).
- Never ask for or print tokens, phone numbers or codes.
- The guide teaches; it does not set up. Setup belongs to the other
  `/tpk:*` skills — point to them.
