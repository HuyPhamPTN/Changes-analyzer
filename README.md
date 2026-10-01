
# changes-analyze
## What it produces

Per run, into `docs/changes/<id>/` of the current repo:

| File | Audience | Content |
|------|----------|---------|
| `business.md` | BA / PO / QA | Plain-language: what changed, why, who's affected, UI affected, risk + rollback, QA checklist. Zero code. |
| `code.md` | Dev team | **High-level merge-conflict report** — what collides when the two branches merge, conflict type, competing intent, resolution direction. NOT a diff restatement (that's in the PR). |
| `ui-affected.md` | QA / BA | Screen + route map with QA click-paths; regression-test list. |

Plus one row appended to `docs/CHANGES.md` (living index).

No diagrams. Read-only by default (never commits/pushes unless `--commit`).

## Install — Claude Code

Global (all repos):
```
~/.claude/skills/changes-analyze/SKILL.md
```
Or per-repo:
```
<repo>/.claude/skills/changes-analyze/SKILL.md
```
Restart the session. Invoke:
```
/changes-analyze <PR>
/changes-analyze main compare with origin/main
```

## Install — Cursor

**Requires Cursor 1.6+** (custom slash commands landed in 1.6). Check: Cursor → Help → About. Older → update first, else the command never appears.

Cursor loads slash commands from `.cursor/commands/*.md` in the **folder you actually open in Cursor** (the project root), or from the global `~/.cursor/commands/`. It does **not** search subfolders. So the file must sit in the root you open — not in a sibling/tooling folder.

Pick the scope that matches how you open Cursor:

**A. One specific repo (most reliable)** — you open e.g. `premium` in Cursor:
```bash
mkdir -p "<repo>/.cursor/commands"
cp changes-analyze.md "<repo>/.cursor/commands/changes-analyze.md"
```



**B. Every Cursor project (global)**:
```bash
mkdir -p ~/.cursor/commands
cp changes-analyze.md ~/.cursor/commands/changes-analyze.md
```

Then: open the Cursor **Agent** chat, type `/` → pick `changes-analyze` (or type `/changes-analyze main compare with origin/main`).


### Troubleshooting
- Command not in the `/` list → wrong location (see above) or Cursor < 1.6.
- Still missing after copy → reload window (`Ctrl+Shift+P` → "Reload Window").
- Type `/` in the **Agent** panel, not inline edit.
- Confirm the file: `.cursor/commands/changes-analyze.md` starts with a `--- description: ... ---` frontmatter block.

> Claude Code vs Cursor files are **mirrors** — Cursor can't read `~/.claude/skills`, so each is self-contained. Edit both when you tune the Ecosystem Map.

## Usage

```
/changes-analyze <PR>                          # PR vs its base branch
/changes-analyze <PR> compare with main        # PR head vs main
/changes-analyze <PR> compare with <PR2>       # two PRs
/changes-analyze <branch> compare with main    # branch vs ref
/changes-analyze <sha1> compare with <sha2>    # two commits/tags
/changes-analyze <PR> --commit                 # also git-commit the dossier (never pushes)
```
`<PR>` = number (uses `gh`). Branch/tag/sha used directly with `git`. Default compare target = `main`.

Run **inside the target repo** (premium / reserve / employer / my-injuryex).

## How the conflict report works

Uses `git merge-tree --write-tree <target> <source>` (Git 2.38+) to **predict conflicts without merging** — no pull, no working-tree change. Conflict types: `content`, `rename/delete`, `rename/rename`, `add/add`, `modify/delete`, `binary`, `submodule`.

- Branches **diverged** (both sides have commits) + same-region edits → populated conflict table.
- One side an **ancestor** (behind/ahead only) → fast-forward, 0 conflicts (short form).
- Uncommitted local edits overlapping incoming → flagged as dirty-worktree block.

You do **not** pull to preview. `git fetch` + merge-tree is enough.


## Files in this package

```
SKILL.md              → Claude Code   (~/.claude/skills/changes-analyze/SKILL.md)
changes-analyze.md    → Cursor        (<repo>/.cursor/commands/changes-analyze.md)
README.md             → this file
```


