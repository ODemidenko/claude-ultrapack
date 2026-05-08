# Kebab-case Slugs and `docs/RFCs` Folder Rename

**Status:** planning
**Branch:** main
**Worktree:** none
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
<empty — filled by up:uplan>

## Verify
<empty — filled by up:uverify>

## Conclusion
<empty — filled by up:ureview>

### Hands-off decisions
- make: size = Medium — two distinct mechanical changes touching ≥10 files across skills/commands/agents; not a one-line trivial edit, so default Medium per hands-off rules.
- make: task-file location = `docs/tasks/slugs-and-rfcs.md` — rigid path under the current (pre-change) rule; do not pre-apply the very change being designed.
- udesign: hard rename, no dual-read fallback or tombstone — single-user fork; no external consumers warrant graceful deprecation.
- udesign: kebab rule codified in `/up:make` only, not in `_principles.md` — naming conventions are out-of-domain for GPC1–GPC8 (software design); restating per-skill would violate GPC8 (DRY).
- udesign: kebab scope = all files the workflow creates (today: task files via `/up:make`, summary task files via `/up:summary`) — matches user's literal "all the created file names"; no practical cost since today there is only one created-file type.

### Deferred (needs user input)
<empty — populated only when Mode is hands-off and a choice had no conservative default>
