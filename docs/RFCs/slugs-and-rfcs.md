# Kebab-case Slugs and `docs/RFCs` Folder Rename

**Status:** executing
**Branch:** task/slugs-and-rfcs
**Worktree:** .worktrees/task/slugs-and-rfcs
**Mode:** hands-off

## Design

Rename the task-file home from `docs/tasks/` to `docs/RFCs/`, and lift the kebab-case naming rule from an implicit slug step in `/up:make` to an explicit, normative rule covering both slugs and any file the workflow creates. Both changes are mechanical text edits across 12 plugin/repo files plus a `git mv` of 8 done task files (and this in-flight one). No behavior changes downstream.

**Scope (the literal edits):**
1. Replace 24 occurrences of `docs/tasks` with `docs/RFCs` across `plugins/up/{commands,agents,skills}/`, plus repo-root `CLAUDE.md` and `README.md`. Verified all are plain path strings — no globs, no variable substitution, no dynamic construction.
2. `git mv docs/tasks/* docs/RFCs/` for the 8 existing done task files (preserves `git log --follow`). The in-flight task file `slugs-and-rfcs.md` moves with them in the same commit, after its content is written.
3. Strengthen the kebab rule in `plugins/up/commands/make.md`: today step 1 says "Derive a kebab-case slug" only for the slug. Extend to a one-line explicit rule covering "any file this workflow creates uses kebab-case", grep-able and citable from other skills.

**Out of scope:**
- Adding the kebab rule to `plugins/up/skills/_principles.md`. GPC1–GPC8 are software-design invariants; a directory-naming convention is out-of-domain there and would muddy the principles file.
- Backwards-compat shims (dual-reading both paths, tombstone redirects). Single-user fork, no external consumers — hard rename is right.
- Renaming any other directory under `docs/`. Only the `tasks` subfolder is in scope per user phrasing.

**Decisions and tradeoffs:**
- **Hard rename in one commit.** All 24 references and the `git mv` ship together. Tradeoff: one large diff vs. zero intermediate state where some skills point at `docs/tasks/` and others at `docs/RFCs/`. Reverting is one `git revert`. Per-file PRs were rejected because skills hand off across files — any partial state breaks the workflow.
- **Codify in `/up:make`, no new principles file.** `/up:make` is the orchestrator that creates files; `summary.md` is the only other file-creating site (already kebab-cases its slugs). One sentence in `/up:make` is enough; a new doc would be over-engineering for one rule.

TDD: no (repo is doc-only per CLAUDE.md design principle: "no runtime code, no unit tests; verification is install-and-invoke").

### Invariants
- IV1 — After the change, no tracked file in the repo contains the literal string `docs/tasks`. Verifiable via `git grep -F 'docs/tasks'` returning empty.
- IV2 — All task files (this one, the 8 done files, any future ones) live at `docs/RFCs/<kebab-slug>.md`. No task file remains under `docs/tasks/`; the directory ceases to exist.
- IV3 — `plugins/up/commands/make.md` contains an explicit, grep-able rule that both task slugs and any file the workflow creates use kebab-case.

### Assumptions
- AS1 — The recon's 24 literal occurrences across 12 files are the complete set; no skill computes the path dynamically. Confirmed by a follow-up grep for `${…}tasks`, `join…tasks`, `path…tasks` patterns — empty. Conclusion must report final `git grep -F 'docs/tasks'` is empty.
- AS2 — `git mv` preserves `git log --follow` history for the 8 existing task files (standard git behavior). Conclusion confirms on at least one moved file.
- AS3 — User reruns `/plugin install up@ultrapack` after merge so Claude Code's plugin cache reloads the edited skills (per CLAUDE.md "Local development" reminder). Without that, the change exists on disk but isn't loaded into running sessions.

### Unknowns
- UK1 — Whether uncommitted work in another worktree (e.g. an unfinished task file) sits under `docs/tasks/` and would be missed by the move. uexecute checks `git worktree list` and the worktree's own `git status` before the `git mv` and surfaces any hits.

## Plan

Approach: split into two phases — content edits (PH1: 25 string replacements + IV3 wording in `/up:make`) then file moves (PH2: `git mv` of 9 task files). Two phases give a clean rollback boundary per change-type.

### PH1 — Content edits

- **1.1** `plugins/up/commands/make.md` (modify) — 4 path replacements (lines 7, 23, 32, 36). Plus IV3: append a sentence to `### 1. Slug` codifying that both the slug and any file the workflow creates use kebab-case.
  - Respects: IV1, IV3
- **1.2** `plugins/up/commands/summary.md` (modify) — 4 path replacements (lines 26, 47, 65, 66).
  - Respects: IV1
- **1.3** `plugins/up/agents/reviewer.md` (modify) — 1 path replacement (line 12).
  - Respects: IV1
- **1.4** `plugins/up/skills/_brevity.md` (modify) — 1 path replacement (line 3).
  - Respects: IV1
- **1.5** `plugins/up/skills/uverify/SKILL.md` (modify) — 1 path replacement (line 79).
  - Respects: IV1
- **1.6** `plugins/up/skills/ureview/SKILL.md` (modify) — 2 path replacements (lines 46, 57).
  - Respects: IV1
- **1.7** `plugins/up/skills/udocument/SKILL.md` (modify) — 1 path replacement (line 48).
  - Respects: IV1
- **1.8** `plugins/up/skills/udesign/SKILL.md` (modify) — 4 path replacements (lines 8, 40, 41, 42). Lines 40–42 are illustrative slugs inside a `<good-example>` block; replace anyway to keep examples consistent with the new convention.
  - Respects: IV1
- **1.9** `plugins/up/skills/uplan/SKILL.md` (modify) — 2 path replacements (lines 8, 31).
  - Respects: IV1
- **1.10** `plugins/up/skills/uexecute/SKILL.md` (modify) — 1 path replacement (line 8).
  - Respects: IV1
- **1.11** `CLAUDE.md` (modify) — 1 path replacement (line 10).
  - Respects: IV1
- **1.12** `README.md` (modify) — 3 path replacements (lines 13, 25, 48 in the worktree; main-side numbers differ by +2 because of the user's uncommitted fork-credit line — match by content, not number).
  - Respects: IV1
- **1.13** Verify gate before commit: `git grep -F 'docs/tasks'` from the worktree returns empty in tracked file content. Resolves IV1 at content level (file paths still pending PH2).
- Implementer: `up:implementer-sonnet` — pure mechanical text replacement plus a one-sentence addition.
- Commit: `docs: rename docs/tasks → docs/RFCs in plugin/repo references; codify kebab-case file rule`

### PH2 — Move task files

- **2.1** Preflight: `git status --porcelain docs/tasks/` and `git worktree list`. If any uncommitted file appears under `docs/tasks/` other than the 9 expected, stop and log to `### Deferred (needs user input)`. Resolves UK1.
- **2.2** `mkdir -p docs/RFCs`.
- **2.3** `git mv docs/tasks/*.md docs/RFCs/` — 9 files (8 done + this in-flight one).
- **2.4** `rmdir docs/tasks` (must succeed; failure means a stray non-`.md` file remains — stop and inspect).
- **2.5** Verify: `git log --follow --oneline docs/RFCs/ultrapack-v1.md | head` shows pre-move commits. Confirms AS2.
- **2.6** Verify: `[ ! -d docs/tasks ]` true and `ls docs/RFCs/*.md | wc -l` equals 9. Confirms IV2.
- Implementer: `up:implementer-sonnet` — pure file rename via `git mv`.
- Commit: `docs: git mv docs/tasks/*.md → docs/RFCs/`

### Risks / rollback
- RK1 — Forgotten dynamic reference outside the 25 enumerated lines. Mitigation: PH1.13 grep gate; AS1's design-time check for template/computed forms returned empty. Both must hold.
- RK2 — Merge conflict against the user's pre-existing uncommitted edits on main (`CLAUDE.md` line 10, `README.md` near fork-credit). Already logged in `### Hands-off decisions`. Resolution is a manual merge; not blocking the task.
- RK3 — `udesign/SKILL.md` lines 40–42 are illustrative slugs in a `<good-example>` block; replacing them updates published example *text* but not behavior. In scope per IV1.

Rollback: `git revert <PH1-sha>` undoes content edits; `git revert <PH2-sha>` reverses the moves.

## Verify

**Result:** passed

Positive:
- CK1 — PH1 commit `3227989` shows 12 files modified, 26/26 inserts/deletes
- CK2 — PH2 commit `5ff2b72` shows 9 renames (`docs/{tasks => RFCs}/...`), 0/0 line changes
- CK3 — kebab rule present at `plugins/up/commands/make.md:19`
- CK4 — `make.md` and `udesign/SKILL.md` now reference `docs/RFCs/<slug>.md`

Negative:
- CK5 — `docs/tasks/` directory absent; `git ls-files 'docs/tasks/*'` returns 0 entries
- CK6 — `git grep -nF 'docs/tasks' -- ':!docs/RFCs/*'` empty (normative file set clean)

Invariants / assumptions:
- CK7 (IV2) — `ls docs/RFCs/*.md | wc -l` = 9; `[ ! -d docs/tasks ]` true
- CK8 (IV3) — kebab rule sentence covers both the slug and any file the workflow creates
- CK9 (IV1, scoped) — holds for normative files; historical task narratives exempt per `### Deviations from plan`
- CK10 (AS2) — `git log --follow` on `docs/RFCs/ultrapack-v1.md` shows pre-move `d86bd96`; on `docs/RFCs/interface-first-parallel.md` shows pre-move `1494ea4`

Notes: end-to-end smoke (install-and-invoke) deferred — see `### Deferred (needs user input)`.

## Conclusion
<empty — filled by up:ureview>

### Deviations from plan
- IV1 verify gate scope clarification (PH1.13): `git grep -F 'docs/tasks'` is empty across the 12 normative files (skills, commands, agents, `CLAUDE.md`, `README.md`), but 5 hits remain inside `docs/tasks/*.md` task narratives (this file's Design + 2 older tasks legitimately referencing the old path as history). Treating IV1 as satisfied for normative files; historical task-file narrative is out of scope per `_brevity.md` principle 3 (the file is the record). PH2's `git mv` moves these files to `docs/RFCs/` but the in-content references stay; this is correct behavior.

### Hands-off decisions
- make: size = Medium — two distinct mechanical changes touching ≥10 files across skills/commands/agents; not a one-line trivial edit, so default Medium per hands-off rules.
- make: task-file location = `docs/tasks/slugs-and-rfcs.md` — rigid path under the current (pre-change) rule; do not pre-apply the very change being designed.
- udesign: hard rename, no dual-read fallback or tombstone — single-user fork; no external consumers warrant graceful deprecation.
- udesign: kebab rule codified in `/up:make` only, not in `_principles.md` — naming conventions are out-of-domain for GPC1–GPC8 (software design); restating per-skill would violate GPC8 (DRY).
- udesign: kebab scope = all files the workflow creates (today: task files via `/up:make`, summary task files via `/up:summary`) — matches user's literal "all the created file names"; no practical cost since today there is only one created-file type.
- make: branch=`task/slugs-and-rfcs`, worktree=`.worktrees/task/slugs-and-rfcs` — hands-off mandates dedicated branch + worktree (never edit `main` directly). `.worktrees/` already gitignored.
- make: did not commit user's pre-existing uncommitted edits on main (`marketplace.json`, `CLAUDE.md`, `README.md`, `plugin.json`) — they belong to a separate authorship/install-reminder change made before `/up:make`. Side effect: merging this branch may produce a small conflict on the `docs/tasks` lines in `CLAUDE.md`/`README.md`; resolvable in seconds.
- uplan: plan auto-approved (hands-off) — two-phase content/rename split; both phases dispatch to `up:implementer-sonnet` (pure mechanical edits and `git mv`).

### Deferred (needs user input)
- End-to-end smoke (install-and-invoke): from this worktree the rebuilt plugin can't be loaded — Claude Code reads `~/.claude/plugins/cache/ultrapack/up/<ver>/`, not the source. After merge, rerun `/plugin install up@ultrapack`, invoke `/up:make` on a throwaway slug, confirm the new task file lands at `docs/RFCs/<slug>.md` (not `docs/tasks/`).
