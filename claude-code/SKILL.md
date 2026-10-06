---
name: changes-analyze
description: "Generate a cross-app Change Dossier for a PR or ref comparison in the InjuryEX ecosystem (Premium, Reserve, Employer, MyInjuryEX, Core). Use for /changes-analyze, 'analyze this PR', 'change dossier', 'what will conflict if I merge', or comparing a PR/branch against main or another PR. Produces business.md (BA/PO/QA plain-language change summary), code.md (high-level branch-conflict report for devs — what will collide on merge, not a diff restatement), and a UI-affected screen map. Read-only unless --commit."
---

# /changes-analyze

Turn a PR or ref-diff into a **Change Dossier**: a plain-language change summary (business.md), a **high-level branch-conflict report** (code.md — what will collide when these two branches merge, so devs can go resolve the detail), and a **UI-affected** screen/route map. Also flags which OTHER apps in the InjuryEX ecosystem might break. Built for a polyrepo where the PO vibe-codes fast and Dev/QA/BA need to understand change, merge risk, and impact without reverse-engineering the UI.

**code.md is NOT a diff restatement** — file lists, line counts, and per-file churn already live in the GitHub PR. code.md answers one question the PR does not: *if I merge branch A into branch B, what conflicts, how bad, and which way should it resolve* — at a level a dev team acts on, not line-by-line.

## Usage

```
/changes-analyze <PR>                              # PR vs its own base branch
/changes-analyze <PR> compare with main            # PR head vs main
/changes-analyze <PR> compare with <PR2>           # two PRs against each other
/changes-analyze <branch> compare with main        # arbitrary branch vs ref
/changes-analyze <sha1> compare with <sha2>        # two commits/tags
/changes-analyze <PR> --commit                     # also git add+commit the dossier (never pushes)
```

`<PR>` = a number (uses `gh`). A branch, tag, or sha is used directly with `git`. Compare target defaults to `main` when omitted, or to the PR's own base branch in bare `<PR>` form.

## Procedure

### 1. Resolve repo + app
Run in the current repo (cwd). Identify the app from the git remote / folder name:
`premium` · `reserve` · `employer` · `my-injuryex` (MIE) · `core`. Selects the right rows in the **Ecosystem Map** + **UI Map** below.

### 2. Resolve refs + diff
- **PR number:** `gh pr view <PR> --json number,title,url,headRefName,baseRefName,author,state,createdAt,mergedAt`
  - bare form: `gh pr diff <PR>` (PR vs its base)
  - `compare with <X>`: resolve `headRefName` of `<PR>`, then `git fetch origin` and `git diff origin/<X>...<headRefName>`
- **Branch/sha:** `git diff <target>...<source>` (three-dot = changes on source since divergence).
- Always also capture:
  - `git diff --stat <base> <head>` (file list + churn)
  - For the largest / new files, read the actual added code (`git show <head>:<file> | head -40`). Never write a dossier from filenames alone.
- **Timestamps (date + time, always with UTC offset).** Format every timestamp as `YYYY-MM-DD HH:mm (UTC±hh:mm)`, e.g. `2026-10-06 14:35 (UTC+07:00)`. Never write a date without a time.
  - **Generated** (when this dossier was produced): `date "+%Y-%m-%d %H:%M (UTC%:z)"` (PowerShell: `Get-Date -Format "yyyy-MM-dd HH:mm '(UTC'zzz')'"`).
  - **Source / target last commit:** `git log -1 --format=%cd --date=format:"%Y-%m-%d %H:%M (UTC%z)" <ref>` (insert the colon in the offset: `+0700` → `+07:00`).
  - **Merge-base commit:** same command on `$(git merge-base <target> <source>)`.
  - **PR merged at:** `mergedAt` from `gh pr view` (ISO UTC, e.g. `2026-10-06T07:35:12Z`) → convert to local time with offset. If `state` ≠ `MERGED` write `not merged yet`. Branch/sha mode with no PR → `n/a`.

### 2b. Detect branch conflicts (code.md core)
Predict what will conflict when `<source>` merges into `<target>` — **without merging**:
- Preferred (Git 2.38+): `git merge-tree --write-tree --name-only <target> <source>`. Exit/first line = tree OID; a conflict list follows. Conflict-type vocabulary: `CONFLICT (content)`, `CONFLICT (rename/delete)`, `CONFLICT (rename/rename)`, `CONFLICT (add/add)`, `CONFLICT (modify/delete)`, `CONFLICT (binary)`, `CONFLICT (submodule)`. Clean merge → no conflict records.
- Fallback (older git): `git merge-tree <merge-base> <target> <source>` and scan for `<<<<<<<` / `changed in both`, or a dry `git merge --no-commit --no-ff <source>` then **always `git merge --abort`** (never leave a merge state).
- For each conflicting file, capture the **competing intent** (why each side touched it), high-level: `git log --oneline <merge-base>..<target> -- <file>` and `... <merge-base>..<source> -- <file>` + the top hunk region (function/section names, not full lines).
- If `<source>` is a strict ancestor/descendant (fast-forward, 0 divergence) → **no conflicts**; code.md says "clean fast-forward" and lists the incoming changes at a heading level only.
Keep it high-level: name the file, conflict type, what each branch wanted, the collision area, a resolution direction + owner. The dev resolves the actual lines.

### 3. Map UI affected (where the change shows up)
From the changed files, resolve the **UI surfaces** touched — the screens/routes a human sees — so QA knows exactly where to click. Detect via:
- Changed **route files** → the route path (see UI Map for each app's route root).
- Changed **components** → grep which routes import them: `grep -rl "<ComponentName>" src/routes` (or MIE `app/`). Roll up to the user-facing screen.
- Changed **models/utils with no UI file** → find the screen that consumes them the same way, else mark "backend/logic only — no direct UI".
Output: screen name + route path + what changed there + a QA click-path.

### 4. Classify against the Ecosystem Map (the two gates)
- **Gate 1 — contract break:** did the diff change any **Contract file** WITHOUT a version bump? Yes → 🔴 blocking risk. Contract only *imported* (not edited) → PASS, say so.
- **Gate 2 — shared surface:** touches any **Shared surface** (API routes, Core link, cross-app models, WPI/Dayforce)? Yes → deep cross-app impact section in code.md (Cross-app consumers table). No → lightweight dossier, note the app is self-contained.

### 5. Emit the dossier
Write to `docs/changes/<id>/` in the current repo, `<id>` = `IE-####` from branch name if present, else `PR-<n>`, else short sha:
```
docs/changes/<id>/business.md
docs/changes/<id>/code.md
docs/changes/<id>/ui-affected.md     # screen/route map + QA click-paths
```
Append one row to `docs/CHANGES.md` (create if missing): `| <id> | <app> | <title> | <risk> | <generated YYYY-MM-DD HH:mm (UTC±hh:mm)> | <merged YYYY-MM-DD HH:mm (UTC±hh:mm) / not merged yet> | link |`.

### 6. Do not commit or push
Default = write files only. Only with `--commit`: `git add docs/changes/<id> docs/CHANGES.md && git commit`. **Never `git push`.** Never touch app source.

---

## Ecosystem Map (cross-app knowledge — the brain)

If `docs/ecosystem-map.md` exists at repo/workspace root, read it first and prefer it. Otherwise defaults:

**Contract files (Gate 1 watch — change here needs a version bump):**
| App | Contract / shared-surface files |
|-----|-------------------------------|
| premium | `src/models/treatment-booking-contract.ts`, `src/routes/api/v1/peme/bookings/**`, `src/routes/api/v1/**` |
| reserve | LMS integration adapters (Clio/ActionStep/LEAP), booking sync client |
| employer | Dayforce integration adapters, health-manager booking client |
| my-injuryex | `services/premium/**`, `services/peme/**`, `services/booking/**` (HTTP clients — consumers, not producers) |
| core | patient + booking canonical schema (shared DB) |

**Shared surfaces → consumers (Gate 2 impact routing):**
| Producer surface | Consumed by | Default risk note |
|------------------|-------------|-------------------|
| Premium `api/v1/peme/bookings/**` (booking HTTP API) | MIE `services/premium/**`, `services/peme/**`; Employer booking client | Field/endpoint change = silent break in MIE/Employer clients (string-URL boundary, no type link). |
| Premium / Core **patient identity + link** (`email-tray-core-create.ts`, `email-tray-existing-store.ts`) | Core DB; MIE `services/premium/patient-profile`, `arrival-checkin` | Patient-id shape change breaks MIE arrival/check-in resolution. |
| **WPI** extraction/data (`referral-drop-wpi-*`) | Reserve medico-legal matters | No code link today → schema-match review; WPI feeds legal claims. |
| Employer **Dayforce** workforce/booking sync | Employer health-manager; Core | Auth scheme / field change breaks tenant sync. |
| Booking status/event model (`treatment-booking-contract.ts`) | All apps on booking workflow | Any status/enum change ripples ecosystem-wide → highest risk. |

**Key fact:** most cross-app edges are **HTTP string-URL boundaries across separate repos**, so no code-graph tool (CodeGraph/graphify) follows them automatically. This map IS the substitute join. Keep it current.

## UI Map (route roots per app — for the UI-affected section)
| App | Route/screen root | Notes |
|-----|-------------------|-------|
| premium | `src/routes/**` (TanStack file routes; `_authenticated/` = signed-in). `.tsx` path ≈ URL. | e.g. `src/routes/_authenticated/doctor/pending.tsx` → `/doctor/pending`. |
| reserve | `src/routes/**` | LMS-facing screens. |
| employer | `src/routes/**` | Health-manager / Dayforce screens. |
| my-injuryex | `app/**` (Expo Router; file path ≈ screen). | Mobile screens. |
| core | n/a (backend) | Usually no direct UI. |

---

## Templates

### business.md (audience: BA / PO / QA — plain language, no code)
```markdown
# Change Dossier — Business

**Change:** <PR/ref> · `<branch>` · <App>   **Compared against:** <base>
**Generated:** <YYYY-MM-DD HH:mm (UTC±hh:mm)> · **Author:** <author>
**Source last commit:** <YYYY-MM-DD HH:mm (UTC±hh:mm)> · **PR merged:** <YYYY-MM-DD HH:mm (UTC±hh:mm) / not merged yet / n/a>
**Apps affected:** <list with role>   **Risk:** 🟢/🟡/🔴 <one-line why>
**Booking contract changed:** Yes/No (<version bump?>)

## What changed (plain language)
<numbered, one paragraph per feature, zero jargon>

## Why
<business goals / pain removed>

## Who / what is affected
<table: who | impact>

## UI affected (where to look)
<table: screen | route | what changed | QA click-path>   — summary; full detail in ui-affected.md

## Workflow impact
<one-line flow of the changed user path>

## Risk + rollback
<risk drivers · main failure mode · rollback · migration?>

## QA acceptance checklist
- [ ] <derived from the diff, testable>
```

### ui-affected.md (audience: QA / BA — the screen map)
```markdown
# Change Dossier — UI Affected

**Change:** <PR/ref> · <App>   **Generated:** <YYYY-MM-DD HH:mm (UTC±hh:mm)>

| Screen | Route / file | Changed? | What changed | QA click-path |
|--------|--------------|----------|--------------|---------------|
| <name> | `/path` (`src/routes/...tsx`) | ✏️ updated / ➕ new / ⚠️ indirect | <plain> | <steps to reach + verify> |

## Backend/logic changes with no direct UI
- <util/model> → affects <screen> indirectly via <path>

## Screens to regression-test (not changed, but consume changed code)
- <screen> — <why at risk>
```

### code.md (audience: dev team — HIGH-LEVEL BRANCH-CONFLICT REPORT)
Goal: what collides when `<source>` merges into `<target>`, at a level the dev team acts on. NOT a diff/churn restatement (that is in the PR). If clean, say so in two lines and stop.
```markdown
# Change Dossier — Merge Conflict Report

**Merge:** `<source>` → `<target>`   **App:** <App>   **Merge-base:** `<short-sha>` (<YYYY-MM-DD HH:mm (UTC±hh:mm)>)
**Generated:** <YYYY-MM-DD HH:mm (UTC±hh:mm)>   **Source last commit:** <YYYY-MM-DD HH:mm (UTC±hh:mm)>   **Target last commit:** <YYYY-MM-DD HH:mm (UTC±hh:mm)>   **PR merged:** <YYYY-MM-DD HH:mm (UTC±hh:mm) / not merged yet / n/a>
**Mergeable:** ✅ clean / ⚠️ <N> conflicts / ⏩ fast-forward (no divergence)
**Detected by:** `git merge-tree --write-tree <target> <source>` (no merge performed)
**Overall merge risk:** 🟢/🟡/🔴 <one line>

## Conflicts (high level)
| File | Type | `<source>` intent | `<target>` intent | Collision area | Risk | Resolution direction | Owner |
|------|------|-------------------|-------------------|----------------|------|----------------------|-------|
| `path` | content / rename-delete / add-add / modify-delete / binary | <what your branch did> | <what target did> | <function/section, not lines> | 🟢/🟡/🔴 | keep source / keep target / merge both / needs discussion | <team/person> |

## Clean but semantically risky (auto-merges, may still break)
- `path` — both branches touched <module>; git merges textually but behavior may clash. Re-test <what>.

## Contract / cross-app conflicts (Gate 1 + Gate 2)
- **Gate 1:** does any conflicting file = a Contract file? → 🔴 escalate (booking contract collision = ecosystem-wide).
- **Gate 2:** conflicting shared surface → which other apps to re-verify (from Ecosystem Map).

## Suggested resolution plan (order)
1. <file/group> — <direction> — <why> — re-run <tests>
2. ...
(High level only. Dev team resolves actual hunks.)
```
When fast-forward / no conflicts:
```markdown
# Change Dossier — Merge Conflict Report

**Merge:** `<source>` → `<target>`   **Mergeable:** ⏩ fast-forward, 0 conflicts   **Generated:** <YYYY-MM-DD HH:mm (UTC±hh:mm)>
`<source>` is <N> behind / ahead; no divergent edits. Nothing to resolve — `git merge`/`git pull` applies cleanly.
Incoming (heading level only): <bullet list of themes>.
```

## Rules
- Real analysis only: read actual diffs + new-file headers, and grep components → routes for the UI map. No hallucinated files/screens.
- business.md = zero code identifiers. ui-affected.md = screen names a QA recognizes + exact route.
- code.md = high-level MERGE-CONFLICT report from `git merge-tree` — what collides, conflict type, competing intent, resolution direction. NOT a diff/file-churn restatement (that's in the PR). Clean merge → two lines, stop.
- Never leave a merge state: if a dry `git merge` is used as fallback, always `git merge --abort` after.
- No diagrams. Describe flow + cross-app impact in prose/tables inside the docs.
- Read-only by default. Never push. `--commit` writes only the dossier files.
- Every date is a timestamp: `YYYY-MM-DD HH:mm (UTC±hh:mm)`, taken from `date` / `git log` / `gh` output — never estimated. Missing value → `n/a`, not a guess.
- Keep each doc tight; a dossier is a decision aid, not a spec.
