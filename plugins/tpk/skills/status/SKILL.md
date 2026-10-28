---
name: status
description: >
  Print a quick status digest of the user's TesterPayKit setup —
  who's logged in, which projects exist, recent sessions, open bug
  submissions, and the recruiting pipeline. Trigger when the user says
  "tpk status", "what's in testerpaykit", "show me my TPK projects",
  "any new bug reports", "what's the state of TPK", or asks "where are
  we" inside a TPK context. Read-only — does not modify any TPK state.
---

# TesterPayKit status digest

Goal: in ~5 seconds, surface enough state that the user knows where
they stand in the TPK ecosystem without clicking through the web
console.

## Steps

1. **Identity.**
   ```bash
   tpk --json whoami 2>&1
   ```
   - On `not logged in`: tell the user to run `tpk login` (device-auth
     via browser) or trigger the `bootstrap` skill, then stop.

2. **Projects.**
   ```bash
   tpk --json project list 2>&1
   ```
   Pretty-print the project name, bundle-id, UUID, and isActive flag.
   If there are no projects, suggest `bootstrap` and stop.

3. **Recent sessions.**
   ```bash
   tpk --json session list --limit 5
   ```
   Show tester, project, duration / engaged minutes, status.

4. **Open bug submissions.**
   ```bash
   tpk --json bug list --status submitted --limit 5
   ```
   Show id, title, severity, project. If any exist, point at the
   `fix-next-bug` skill — that's the loop this plugin is for.

5. **Recruiting pipeline.**
   ```bash
   tpk --json job list
   ```
   Show which projects are recruiting and their pending-application
   counts. Drill down on request with
   `tpk job applications <project-uuid>`.

6. **Render the digest.**
   Plain-text table or bullet list — short, scannable:
   ```
   tpk status
   --------------------
   logged in: dev@example.com (Jane Dev)

   projects (N):
     myapp   com.example.myapp   594c3ee6-...   active

   sessions (last 5): ...
   open bugs (submitted): 2 → run /tpk:fix-next-bug
   recruiting: myapp — 3 pending applications
   ```

## What this skill does NOT do

- Doesn't refresh tokens — if the JWT is expired, surface the 401 to
  the user and recommend `tpk login` again.
- Doesn't mutate anything — triage and fixes belong to `fix-next-bug`.
