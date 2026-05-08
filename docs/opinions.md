# ultrapack — implicit opinions and project-shape assumptions

The plugin is opinionated. Most of those opinions are not stated as requirements; they're baked into how skills phrase rules, what they assume exists, and what they refuse to do. This document surfaces them so you can decide before installing whether your project's shape fits — or where you'll need to bend.

Each item lists **what is assumed**, **where it's encoded**, and **what bends or breaks** if your project doesn't match.

---

## Tooling and platform

### A1 — Git is mandatory; no other VCS supported

**Where:** every workflow skill (`uexecute`, `uverify`, `ureview`) calls `git` directly; `git-worktrees/SKILL.md` is the entire branch-isolation story; reviewer dispatch passes `BASE_SHA`/`HEAD_SHA`.

**Bends:** Mercurial / Fossil / Pijul projects can't use the chain. The skills hardcode `git rev-parse`, `git show`, `git diff`, `git worktree`. There is no abstraction layer.

### A2 — Trunk branch is named `main` (or `master`)

**Where:** `commands/make.md` template defaults `**Branch:** main`; `handsoff/SKILL.md` says "never edit `main` / `master` directly"; `ureview` computes diffs against `main` by default.

**Bends:** projects using `develop`, `trunk`, `release`, or any other primary-branch name will need per-task overrides in the task file header, and the merge-base logic in `ureview` will need adjustment.

### A3 — Pre-commit hooks are respected, signing is on by default

**Where:** `handsoff/SKILL.md` and the project-level CLAUDE.md forbid `--no-verify`, `--no-gpg-sign`, `-c commit.gpgsign=false`. Implementer agents inherit this.

**Bends:** projects without hooks: harmless. Projects that *expect* hooks to be skipped sometimes (e.g. a slow lint step the CI re-runs): every per-phase commit pays the full hook cost.

### A4 — Conventional Commits style commit messages

**Where:** `agents/implementer.md` mandates `<type>: <concise>` with `feat:`, `fix:`, `refactor:`, `test:`, `docs:`. Same in `implementer-sonnet`.

**Bends:** projects with custom prefixes (`[FEAT]`, ticket IDs, gitmoji) will see the agent fight the convention. Easy to override per task, but the default is wired in.

### A5 — One commit per phase, no squash culture

**Where:** `uexecute/SKILL.md`: "Skip the commit between phases" is in the **Never** list; "Each phase is a coherent commit" in `uplan/SKILL.md`.

**Bends:** projects with squash-only PR policies: granular history is wasted at merge but the plan-diff check still works. Projects that demand single-commit PRs: friction — you'll be amending or rebasing post-execute.

### A6 — Build system is auto-detectable via standard manifests

**Where:** `git-worktrees/SKILL.md` baseline-setup detects only `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`.

**Bends:** Bazel, Buck, Pants, Make-only, custom build scripts, multi-language monorepos — `git-worktrees` will skip baseline setup silently. You'll need to extend the skill or run setup manually.

### A7 — `tmp/` is gitignored and writable

**Where:** `udebug/SKILL.md` and `uverify/SKILL.md` write reproduction scripts to `tmp/`. Style note: "project-local `tmp/` (gitignored); clean up after."

**Bends:** projects using a different scratch dir convention (`scratch/`, `_local/`) need to either alias or accept that ad-hoc files land in `tmp/`.

---

## Repository structure

### B1 — `docs/RFCs/<slug>.md` is the task-file home

**Where:** `commands/make.md` step 2 reads `docs/RFCs/<slug>.md`; every workflow skill writes back to the same file. Resume detection scans `docs/RFCs/*.md`.

**Bends:** if your project keeps RFCs elsewhere (`design/`, `proposals/`, `.specs/`, an external Notion), you must either symlink or accept duplication. The path is hardcoded in skills, not configurable.

### B2 — Snake_case slug naming for task files

**Where:** `commands/make.md` step 1: "snake_case slug, 3 words max." Recent commits in this repo (`d99011a`, `7f5434c`, `2ee3ac2`) renamed everything to enforce this.

**Bends:** existing kebab-case task file collections will need a `git mv` pass.

### B3 — `CLAUDE.md` exists at project root

**Where:** `commands/make.md` "docs refresh" stage and `commands/reflect.md` both update it. `udocument/SKILL.md` treats it as a first-class artifact.

**Bends:** projects without an agent-instruction file get one created. Projects with `AGENTS.md` instead: you'll get *both* unless you redirect.

### B4 — `docs/specs/` is the canonical home for read-only external specs

**Where:** `uexecute/SKILL.md` and `handsoff/SKILL.md`: "External spec files (e.g. anything under `docs/specs/`) are read-only during execute."

**Bends:** projects that mutate specs as part of the work (living specs, executable specs) violate the rule. The skill is opinionated that specs are *inputs*, not artifacts of the task.

### B5 — Worktrees live in `.worktrees/` or `worktrees/`, gitignored

**Where:** `git-worktrees/SKILL.md` priority order; `handsoff` defaults to provisioning one for every hands-off task.

**Bends:** projects that already use a different worktree dir (`.git/worktrees-side/`, `~/work/<repo>/`) need to declare it in `CLAUDE.md`. Projects that don't use worktrees at all: hands-off mode degrades — every task creates one anyway.

### B6 — Single repo, not monorepo-aware

**Where:** all skills assume one `git rev-parse --show-toplevel`. No per-package routing, no workspace detection, no scoping by directory.

**Bends:** monorepos work, but the workflow treats them as a single unit. Per-package CI, per-package versions, per-package CHANGELOGs are not modeled. The reviewer agent diffs the whole repo.

---

## Workflow shape

### C1 — One task at a time per session

**Where:** `commands/make.md` step 2: "if more than one in-flight task, ask which one." `udesign/SKILL.md` scope-check splits multi-feature asks into separate task files.

**Bends:** parallel multi-task development across one repo is unsupported. You can run two `/up:make` sessions concurrently in different worktrees, but each owns its own task file; cross-task coordination is manual.

### C2 — Trunk-based, PR-or-merge endpoint

**Where:** `commands/make.md` step 11: "Merge / open PR / Clean up worktree / Move on." No release-branch flow, no hotfix backport flow.

**Bends:** GitFlow, env-branch (`develop` → `staging` → `prod`), or release-train projects: the workflow ends before your real merge ceremony begins. You'll handle that ceremony manually after `/up:make` finishes.

### C3 — Solo or near-solo development; review is by an agent

**Where:** `ureview/SKILL.md` dispatches the `reviewer` agent as the independent check. Confidence ≥ 80 is the bar.

**Bends:** team projects with mandatory human review still need that human review *after* `/up:make` finishes. The plugin doesn't model code-owners, required reviewers, or multi-author conflict resolution. Treat agent review as pre-PR self-check, not as the merge gate.

### C4 — Tasks are 1–3 days of work; bigger gets split

**Where:** `uplan/SKILL.md` self-review: "If [the plan] exceeds ~1 screen per day of expected work, trim." `udesign/SKILL.md` scope-check: "can a plan for this piece produce working, testable software on its own?"

**Bends:** epic-sized work (multi-week features, large refactors) must be sliced into multiple task files. The dispatcher will not gracefully handle a 30-phase plan.

### C5 — TDD applicability is a per-task binary

**Where:** `udesign/SKILL.md` records `TDD: yes | no (reason)`. `test-driven-development/SKILL.md` skip-conditions explicitly list "training a model", "exploratory data analysis", "one-off scripts", "UI changes."

**Bends:** mixed-mode work (some library code + some experiment code in one task) doesn't fit cleanly. The applicability rule wants a single answer per task.

### C6 — Manual smoke test is feasible from the workspace

**Where:** `uverify/SKILL.md` Phase 3: "Run the shortest full path that exercises the change in its real shape — CLI, curl, browser, training step." Infeasibility logs to `### Deferred (needs user input)` rather than degrading silently.

**Bends:** projects whose smoke tests need staging infra, secrets, or a separate cluster (think: a service that only runs end-to-end inside a VPN'd k8s cluster) will hit Deferred frequently. The plugin doesn't bridge to staging.

---

## Engineering philosophy embedded

The next group are not procedural — they're values the plugin enforces in *every* artifact it writes. If your codebase or team disagrees, expect to fight the agent on every commit.

### D1 — No silent fallbacks, ever

**Where:** `uexecute/SKILL.md` "Forbidden: inventing fallbacks, defaults, or best-effort behavior." `_principles.md` GPC6 "Fail fast, fewest moving parts."

**Bends:** projects that value graceful degradation (consumer apps that prefer a stale value over a 500) will write code the agent rejects. You can override per task, but the default is fail-loud.

### D2 — Explicit interfaces; no hidden config reads

**Where:** `_principles.md` GPC1: "Caller passes every argument. Each constant / config / magic value is defined in exactly one place and threaded explicitly to its use site."

**Bends:** Spring-style dependency injection, Django settings auto-discovery, environment-variable-driven feature flags — all violate the spirit of GPC1. The agent will critique them in review.

### D3 — Layered architecture with no upward imports

**Where:** invariant examples in `udesign/SKILL.md`: "`Dataset` must not import from `training/`." GPC4 "Cross-boundary leaks are a design bug, not a style nit."

**Bends:** flat single-package projects, "utility-bag" architectures, or codebases without enforced module boundaries: invariants come out vague, the reviewer has nothing to check against.

### D4 — Functions ≤ ~10 lines, one concern per unit

**Where:** `_principles.md` GPC2: "Functions >~10 lines are a smell — split unless justified."

**Bends:** numerical / scientific code where a 40-line function is one indivisible algorithm: the agent will propose splits that hurt readability. Override with a Principle in the task file.

### D5 — Brevity over completeness in artifacts

**Where:** `skills/_brevity.md` five principles, included by every writing stage. "Omit, don't fill"; "evidence only on surprise"; "one sentence, not a paragraph."

**Bends:** teams that document for compliance / audit / onboarding need richer artifacts. The plugin will produce minimal task files; you'll need to enrich them post-task or relax the include.

### D6 — Lead-with-why docs, lists over tables, kill stale content

**Where:** `udocument/SKILL.md`. Applies to `README.md`, `docs/**/*.md`, CLAUDE.md, SKILL.md, docstrings, inline comments.

**Bends:** projects with mandated doc templates, or doc cultures that prefer narrative-first prose, will see the agent rewrite toward terse-bullet style.

### D7 — ID-based cross-references in artifacts (IV1, PC2, AS3, UK4, PH5, RK6, IF7, CK8)

**Where:** every workflow skill defines and references entities by short ID. `udesign` owns IV/PC/AS/UK; `uplan` owns PH/RK/IF; `uverify` owns CK.

**Bends:** task files get unreadable to outsiders not familiar with the convention. The skills mitigate by requiring full sentences at first appearance, but you're still buying into a private vocabulary.

### D8 — Codebase fits a small-context trace

**Where:** `agents/explorer.md`: "Stop when 3-5 essential files are enough to understand the feature." Implies features are not deeply distributed.

**Bends:** very large or microservice-style codebases where one feature touches 20 files in 5 services: the explorer agent will return an incomplete map and the implementer will get insufficient context.

---

## What the plugin **does not** assume

Worth stating explicitly so you don't paint yourself into a corner thinking it's required:

- **No CI integration.** The workflow ends locally. No GitHub Actions hooks, no required status checks, no release-pipeline awareness.
- **No issue tracker integration.** Task files are markdown in the repo; no Linear / Jira / GitHub Issues sync. `commands/reflect.md` mentions "memory" but doesn't write to issue trackers.
- **No language requirement.** Skills work over `.py`, `.ts`, `.go`, `.rs`, `.md`, anything text. The opinionation is procedural and structural, not language-specific.
- **No specific test framework.** `uverify` runs whatever command you tell it to.
- **No model lock-in within Claude.** Agents pin models in frontmatter, but the workflow doesn't require a specific Claude version.

---

## Quick-fit checklist

Run through this before adopting `/up:make` on a project. The more "yes"es, the better the fit:

- Git repo with `main` (or you can rename) as trunk?
- Solo or small-team, with PR-or-merge ending the workflow?
- Build via `package.json` / `Cargo.toml` / `pyproject.toml` / `go.mod` (or willing to extend `git-worktrees`)?
- Comfortable with Conventional Commits and one-commit-per-phase?
- OK with `docs/RFCs/<slug>.md` as the task-file home?
- Layered codebase with checkable module boundaries?
- Tasks usually slice into 1–3 days of work?
- Smoke testing feasible from the local workspace?
- Engineering culture that prefers fail-fast over silent fallbacks?
- Documentation culture compatible with terse, why-led, list-over-table style?

If you're answering "no" to three or more, the plugin will be more friction than help unless you fork it.
