---
name: fix-next-bug
description: >
  Close the agentic feedback→fix loop: pick the highest-priority open
  bug for the current project from TesterPayKit, fetch the LLM-ready
  context blob via the tpk CLI, propose + implement a fix, and mark
  the bug fixed so the tester gets a retest notification. Trigger when
  the user says "fix the next bug", "fix next TPK bug", "claude, was
  hat der Tester gemeldet — bau den Fix", "wir haben einen neuen
  Bug-Report", or asks Claude to handle the latest tester feedback.
  This is the core differentiator vs. plain feedback widgets — real
  testers report, Claude fixes, the tester verifies.
---

# Fix the next TesterPayKit bug

Goal: from "shake-to-report submitted by a tester" to "verified fix +
tester notified for retest" in one Claude Code turn, with the tester's
full context fed to the model as structured data — not
screenshot+gut-feel.

## Prerequisites

- `tpk` CLI on PATH (`which tpk` returns a path). If missing, run the
  `cli-install` skill first.
- Logged in (`tpk --json whoami` returns a coordinator). If not, run
  `tpk login`. Cite the `bootstrap` skill if the user has no project
  yet.
- Current directory is the host app's repo root so file edits land in
  the right tree.

## Steps

1. **Discover the project bundle-id.**
   Read the host's `pubspec.yaml` / `Info.plist` / `build.gradle`
   to recover the bundle identifier. If multiple projects could
   match, ask the user which one. Cache the answer in `.tpk/` (next
   to `.git/`) so future runs don't re-ask.

2. **Fetch open bugs.**
   ```bash
   tpk --json bug list \
     --bundle-id <bundle-id> \
     --status submitted \
     --limit 20
   ```
   Sort by severity (critical > high > medium > low) and then by
   `createdAt` (oldest-first, so the user clears the queue, not just
   the latest).

   If the result has zero bugs, surface: "No open bugs for this
   project — queue is clean." Stop. Don't fabricate work.

3. **Pick the top bug, claim it, load context.**
   Mark it as picked up so it leaves the `submitted` queue:
   ```bash
   tpk bug status <top-bug-id> --to accepted
   ```
   (Allowed `--to` values: draft, submitted, underReview, accepted,
   duplicate, rejected, fixed, verified.)
   Then fetch the LLM-ready blob:
   ```bash
   tpk --json bug context <top-bug-id> > /tmp/tpk-bug.json
   ```
   Read `/tmp/tpk-bug.json`. Key fields to anchor on:
   - `bug.title` / `bug.description` — what the tester typed
   - `bug.currentScreen` — where they were when it happened
   - `bug.severity`, `bug.frequency`, `bug.affectedFeature`
   - `repro.fromBreadcrumbs` — last ~20 actions, each with
     `{t, action, screen, widget}`. The widget names map directly
     into the host's source if it uses TPK's `TesterTaskRelevant`
     annotations; otherwise grep for the widget name.
   - `device.platform`, `device.osVersion`, `device.model` — to know
     whether this is a platform-specific bug
   - `app.buildMode` / `app.environment` — debug vs release, local vs prod
   - `agentHint` — one-paragraph templated summary; **always quote
     this verbatim to the user** so they can sanity-check your
     interpretation.

4. **Propose the fix.**
   Use the bug-context to:
   - Find the widget / screen in the host's source (grep for
     `currentScreen`, last breadcrumb `widget`, `affectedFeature`)
   - State a one-sentence hypothesis: "I think this happens because
     [X], and the fix is [Y]."
   - Show the user the planned edit BEFORE making it. Wait for green
     light unless the user explicitly said "auto-fix".

5. **Implement and verify.**
   - Edit the file(s).
   - Run `flutter analyze` (or the host project's equivalent).
   - Run the project's test suite if one exists.

6. **Mark fixed → tester gets the retest.**
   Once the fix is verified locally, close the loop server-side:
   ```bash
   tpk bug fix <bug-id> --version <version-where-it-ships>
   ```
   `--version` is required (e.g. "1.4.2" — read it from pubspec.yaml).
   This is a feature, not bookkeeping: it flips the bug to `fixed`,
   creates a retest assignment, and notifies the reporting tester
   ("your bug was fixed — please verify"), feeding the tester's
   verify/reward chain. Optionally hand the retest to a specific
   tester with `--tester <uuid|code>`.

   If the bug turns out to be invalid instead, close it without a
   retest: `tpk bug status <bug-id> --to rejected` (or
   `--to duplicate`).

7. **Hand back.**
   Print a short summary:
   ```
   Bug #ABC123 — "<title>"
   Hypothesis: <one-line>
   Edited: lib/features/x/y.dart (+12 -3). flutter analyze: clean.
   Marked fixed in <version> → tester notified for retest.
   Next: review the diff + `git commit`, ship the build the version
   refers to.
   ```

## What this skill does NOT do

- **Doesn't auto-commit** — the user reviews the diff first. The
  agentic loop is "tester → Claude proposes → human ships".
- **Doesn't auto-deploy** — shipping the build that contains the fix
  is the user's job; `bug fix --version` only records where it ships.
- **Doesn't fabricate context** — if a field is missing in
  `bug context`, surface that to the user and ask. Hallucinated repro
  steps poison the loop.
- **Doesn't pick low-quality reports.** If the bug has no breadcrumbs
  AND no repro steps AND a 5-word description, surface that and ask
  the user to triage manually. The point of the loop is amplified
  signal, not amplified noise.

## Loop hygiene (long-term)

When this skill is used 50+ times against the same project, the
breadcrumb→source-file mapping becomes a learnable pattern. A future
iteration should cache "widget X lives in file Y line N" in `.tpk/`
so subsequent invocations skip the grep step.
