---
description: Generate a cross-app Change Dossier for a PR or ref comparison in the InjuryEX ecosystem (Premium, Reserve, Employer, MyInjuryEX, Core). Produces business.md (plain-language change summary), code.md (high-level branch-conflict report — what collides on merge, not a diff restatement), ui-affected.md. Read-only unless --commit.
---

# /changes-analyze

Turn a PR or ref-diff into a **Change Dossier**: a plain-language change summary (business.md), a **high-level branch-conflict report** (code.md — what collides when the two branches merge, so devs resolve the detail), and a UI-affected screen/route map. Also flags which OTHER apps might break. For a polyrepo where the PO vibe-codes fast and Dev/QA/BA must understand change, merge risk, and impact without reverse-engineering the UI.

**code.md is NOT a diff restatement** — file lists / line counts already live in the GitHub PR. code.md answers: *if `<source>` merges into `<target>`, what conflicts, how bad, which way to resolve* — at dev-team level, not line-by-line.

## Usage
```
/changes-analyze <PR>                          # PR vs its base branch
/changes-analyze <PR> compare with main        # PR head vs main
/changes-analyze <PR> compare with <PR2>       # two PRs
/changes-analyze <branch|sha> compare with <ref>
/changes-analyze <PR> --commit                 # also git commit the dossier (never pushes)
```
`<PR>` = number (uses `gh`). Branch/tag/sha used directly with `git`. Compare target defaults to `main`, or the PR's own base in bare form.

## Procedure
1. **Repo + app.** Run in cwd repo. Identify app from remote/folder: premium · reserve · employer · my-injuryex (MIE) · core. Selects Ecosystem Map + UI Map rows.
2. **Resolve refs + diff.**
   - PR: `gh pr view <PR> --json number,title,url,headRefName,baseRefName,author,state,createdAt,mergedAt`. Bare: `gh pr diff <PR>`. `compare with <X>`: `git fetch origin && git diff origin/<X>...<headRefName>`.
   - Branch/sha: `git diff <target>...<source>`.
   - Always: `git diff --stat <base> <head>`, and `git show <head>:<file> | head -40` on big/new files. Never analyze from filenames alone.
   - **Timestamps** — always date + time + offset: `YYYY-MM-DD HH:mm (UTC±hh:mm)` (e.g. `2026-10-06 14:35 (UTC+07:00)`). Generated: `date "+%Y-%m-%d %H:%M (UTC%:z)"` (PowerShell: `Get-Date -Format "yyyy-MM-dd HH:mm '(UTC'zzz')'"`). Source/target/merge-base commit: `git log -1 --format=%cd --date=format:"%Y-%m-%d %H:%M (UTC%z)" <ref>` (`+0700`→`+07:00`). PR merged: `mergedAt` from gh (UTC ISO → local + offset); not merged → `not merged yet`; no PR → `n/a`.
2b. **Detect conflicts (code.md core).** Predict collisions when `<source>` merges `<target>`, no merge performed:
   - Preferred (git 2.38+): `git merge-tree --write-tree --name-only <target> <source>` → conflict list. Types: `CONFLICT (content)`, `(rename/delete)`, `(rename/rename)`, `(add/add)`, `(modify/delete)`, `(binary)`, `(submodule)`.
   - Fallback: `git merge-tree <merge-base> <target> <source>` scan for `<<<<<<<`, or dry `git merge --no-commit --no-ff <source>` then **always `git merge --abort`**.
   - Per conflicting file capture competing intent: `git log --oneline <merge-base>..<target> -- <file>` vs `... <merge-base>..<source> -- <file>` + top hunk region (function/section, not lines).
   - Ancestor/descendant (fast-forward, 0 divergence) → no conflicts; code.md = "clean fast-forward" + incoming themes at heading level only.
3. **UI affected.** Resolve screens/routes touched: changed route files → route path; changed components → `grep -rl "<Component>" src/routes` (MIE `app/`) → roll up to screen; logic-only files → the screen that consumes them, else "backend only". Output screen + route + what changed + QA click-path.
4. **Two gates.**
   - **Gate 1 (contract break):** contract file changed WITHOUT version bump → 🔴 blocking. Only imported → PASS (say so).
   - **Gate 2 (shared surface):** touches API routes / Core link / cross-app models / WPI / Dayforce → deep cross-app impact table in code.md. Else lightweight.
5. **Emit** to `docs/changes/<id>/` (`<id>` = IE-#### from branch, else PR-<n>, else short sha): `business.md`, `code.md`, `ui-affected.md`. Append row to `docs/CHANGES.md`: `| <id> | <app> | <title> | <risk> | <generated timestamp> | <merged timestamp / not merged yet> | link |`.
6. **No git side effects** by default. `--commit` = `git add docs/changes/<id> docs/CHANGES.md && git commit`. **Never push.** Never touch app source.

## Ecosystem Map (prefer `docs/ecosystem-map.md` if present)
**Contract files (Gate 1 watch):** premium `src/models/treatment-booking-contract.ts`, `src/routes/api/v1/peme/bookings/**`, `src/routes/api/v1/**` · reserve LMS adapters (Clio/ActionStep/LEAP) + booking sync · employer Dayforce adapters + booking client · my-injuryex `services/premium/**`, `services/peme/**`, `services/booking/**` (consumers) · core canonical patient+booking schema.

**Shared surface → consumers:**
| Producer | Consumed by | Risk |
|---|---|---|
| Premium `api/v1/peme/bookings/**` | MIE `services/premium/**`,`services/peme/**`; Employer booking client | field/endpoint change = silent break (string-URL boundary, no type link) |
| Premium/Core patient link (`email-tray-core-create.ts`, `email-tray-existing-store.ts`) | Core DB; MIE `services/premium/patient-profile`,`arrival-checkin` | patient-id shape change breaks MIE arrival/check-in |
| WPI (`referral-drop-wpi-*`) | Reserve medico-legal | schema-match review; feeds legal claims |
| Employer Dayforce sync | Employer health-manager; Core | auth/field change breaks tenant sync |
| `treatment-booking-contract.ts` status/event model | all booking-workflow apps | enum/status change ripples ecosystem-wide — highest risk |

**Key fact:** cross-app edges are HTTP string-URL boundaries across separate repos → no code-graph tool follows them. This map is the substitute join. Keep current.

## UI Map (route roots)
premium/reserve/employer `src/routes/**` (TanStack file routes, `.tsx` path ≈ URL, `_authenticated/`=signed-in) · my-injuryex `app/**` (Expo Router, path ≈ screen) · core = backend, usually no UI.

## Output contents
- **business.md** — header: Generated · Author · Source last commit · PR merged (all timestamps). Plain language, no code: What changed · Why · Who affected · **UI affected (screen|route|change|click-path)** · Workflow impact · Risk+rollback · QA checklist.
- **ui-affected.md** — header with Generated timestamp, then table: Screen | Route/file | new/updated/indirect | what changed | QA click-path. Plus logic-only-no-UI list, and screens to regression-test.
- **code.md** — HIGH-LEVEL merge-conflict report (NOT diff restatement). Header: `<source>`→`<target>`, merge-base (+ its commit timestamp), Generated / source last commit / target last commit / PR merged timestamps, mergeable (✅clean / ⚠️N conflicts / ⏩fast-forward), detected by `git merge-tree`, overall risk. Then: **Conflicts table** (File | Type | source intent | target intent | collision area | risk | resolution direction | owner) · **Clean-but-semantically-risky** list · **Contract/cross-app conflicts** (Gate 1 escalate if a Contract file conflicts; Gate 2 apps to re-verify) · **Suggested resolution plan** (ordered, high level). Clean/fast-forward → two lines + incoming themes, stop.

## Rules
Real analysis only (read diffs + grep components→routes + `git merge-tree`). business.md zero code identifiers; ui-affected.md screen names a QA recognizes. code.md = what collides on merge, conflict type, competing intent, resolution direction — not file/line churn (that's the PR). Never leave a merge state (dry-merge fallback → `git merge --abort`). No diagrams. Read-only default, never push, `--commit` writes only dossier files. Every date = `YYYY-MM-DD HH:mm (UTC±hh:mm)` from `date`/`git log`/`gh`, never estimated; missing → `n/a`.
