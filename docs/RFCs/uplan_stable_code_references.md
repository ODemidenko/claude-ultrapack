# Stable code references and lifecycle commit prompts

**Status:** planning
**Branch:** main
**Worktree:** none
**Mode:** interactive

## Original description
Context:
Between design and plan there can be a longer break. 
Desired behavior:
Propose user to commit design upon udesign finish, to the current branch, no branch creation at this point.

Context:
between planning and implementation there may be another (strictly orthogonal) changes introduced in the code.
Desired behavior:
Implementation plan must rather tell what functions/classes to fix (stable references), instead of referencing the current line numbers (most likely to get obsolete).
Propose the user to commit the RFC document with the plan in it, before execution.

Verify:
the /up:make must propose to create git worktree and branch only when the execution starts, at /up:uexecute, no earlier, as otherwise code may substantially evolve between the planning and executing phase.

Executor:
Executor must prefer NOT commiting the change, in case of single-phase changes. The goal is to allow the user to validate the code before it was committed.
I.e. trigger implementer with `commit: defer` mode, in case of a single phase plan.
And propose user to review the code, upon completion. 

## Design

**Purpose.** Reduce the cost of time elapsing between ultrapack stages. Code drifts between Design ↔ Plan ↔ Execute, so the workflow must (a) snapshot the spec into git at natural pause points, (b) reference code by stable symbols rather than ephemeral line numbers, (c) defer worktree creation until execution truly starts, (d) let the user eyeball single-phase output before it lands as a commit, and (e) write the spec to disk before asking for review so the user can use the editor of their choice.

**Scope.** Five changes, edited into existing skill / agent / command files. No new skills, no new commands.

1. **End-of-`udesign` commit prompt.** After self-review and user approval of Design, ask `Commit RFC with Design? [Y/n]`. On `y`/empty: stage the RFC file (and any associated `git mv` of the seeding backlog file that is already staged), commit with `docs(rfc): design — <slug>`. No branch creation. On `n`: continue. Hands-off default: yes; log under `### Hands-off decisions`.

2. **Stable references in plans.**
   - `uplan` format requires both **symbol name** (class/method/function) and a line range. The symbol name is authoritative; the line range is an advisory hint that may be stale.
   - `up:implementer` process explicitly locates the symbol by name first; treats the line range as a starting hint only; reports if the symbol no longer exists.
   - End-of-`uplan` commit prompt mirroring `udesign`: `Commit RFC with Plan? [Y/n]`, message `docs(rfc): plan — <slug>`. Hands-off default: yes.

3. **Worktree/branch creation moves into `up:uexecute` start.**
   - `/up:make` step 6 ("Branch & worktree decision") is removed.
   - `up:uexecute` "Before starting" gains a new first step: if task-file `**Branch:**` is `main`/`master` and the task isn't trivial, prompt for branch + worktree via `up:git-worktrees`; update task-file headers; switch to the worktree before dispatching any implementer. Trivial tasks may stay on the current branch.
   - Hands-off: same default as today's step 6 (always create branch+worktree), just at a later moment in the flow.

4. **Single-phase plans defer the commit.**
   - When `## Plan` contains exactly one `### PH<N>` heading and no `### Interface graph`, `up:uexecute` dispatches the implementer with `Commit mode: defer`.
   - After the implementer returns `DONE` with staged files, `up:uexecute` prints a short summary (`git diff --stat --staged`), then asks: `Review the staged changes and approve to commit? [y/n/edit]`. On `y`: commit with implementer's proposed message, then continue to `up:uverify`. On `edit`: pause for user edits, then re-prompt. On `n`: stop and log as deferred.
   - Hands-off default: do **not** defer; dispatch with `Commit mode: self` as today. Log the choice.

5. **Write-before-review in `udesign` and `uplan`.**
   - Both skills must write their proposed section to `docs/RFCs/<slug>.md` *before* asking the user for approval. The chat message that requests approval names the file path; the user reviews in their editor of choice (IDE, less, browser) and then says yes/no/changes in chat.
   - `udesign`: re-order steps so "write to task file" precedes "present and wait for approval". The presentation message becomes a short "I've written the Design section to `<path>` — please review and approve, request changes, or reject."
   - `uplan`: same flip — write `## Plan` first, then ask for approval pointing at the file.
   - Iteration: when the user requests changes, the skill edits the file in place and re-prompts pointing to the same file path.
   - Hands-off: skills already auto-approve; the write-first rule still applies (no functional difference, but keeps the invariant uniform).

**Key decisions / tradeoffs.**
- *Symbol-name + advisory line range* (vs. drop line numbers entirely): the line range still helps the implementer (and reviewers) find the right region quickly; making it advisory removes the staleness failure mode without giving up the navigation aid. Cost: one extra self-review bullet in `uplan`.
- *Commit prompt runs the commit* (vs. print the command): matches the project's existing prompt-and-execute pattern; the RFC file is the single staged artifact so blast radius is tiny.
- *Worktree decision in `uexecute`* (vs. `/up:make` step before execute): keeps the contract local to where it matters and means `up:uexecute` is correct even when invoked standalone or on resume.
- *Solo-phase defer triggered by phase count* (vs. user flag): the planner already knows the phase count; no new flag surface. Multi-phase plans behave exactly as today.
- *Write-before-review uses the same file the skill already owns* (vs. a separate review artifact): zero new files, the editor experience just works, and the chat round-trip carries only the approval decision instead of a wall of markdown.

TDD: no (reason: changes are entirely in skill / agent / command markdown files; no reusable runtime logic; verification is install-and-invoke).

**Backwards-compat.**
- The one in-flight task (`docs/RFCs/no_shorthand_references.md`, status `planning`) is unaffected — its Plan is already written under the old format, and on resume `up:uexecute`'s new worktree-prompt step is additive (it sees Branch=main and asks).
- Existing closed task files keep their old-format Plans; nothing rewrites historical RFCs.
- `Commit mode: defer` already exists in the implementer protocol — no new mechanism, just a new trigger condition.

### Invariants

- **IV1 — symbol-name-required-in-plan-bullets** — Every `## Plan` per-file bullet that names a code change must include the affected class / method / function name. Why: line ranges drift between planning and executing; the symbol is the stable handle the implementer resolves against.
- **IV2 — worktree-decision-not-before-execute** — `/up:make` and child skills must not create a branch or worktree before `up:uexecute` is invoked. Why: code can change materially between plan and execute; capturing the worktree earlier captures a stale snapshot.
- **IV3 — no-auto-commit-without-prompt-in-interactive** — End-of-stage commits in interactive mode require an explicit user `y`. Why: project-wide convention; the user stays in control of git history.
- **IV4 — solo-phase-defers-commit-in-interactive** — When `## Plan` has exactly one phase and the mode is interactive, `up:uexecute` must dispatch the implementer with `Commit mode: defer` and pause for user approval before committing. Why: lets the user validate uncommitted output, the core goal of the task.
- **IV5 — write-before-review** — `udesign` and `uplan` must persist their proposed section to the task file before asking the user to review it. Why: lets the user review in any editor instead of being forced to read a long markdown block in chat.

### Principles

- **PC1 — line-range-is-advisory-symbol-is-authoritative** — When a plan bullet carries both, the implementer locates by symbol first; the line range is a hint only. If the symbol is gone, report — do not guess from the line range.
- **PC2 — additive-skill-changes-only** — Edit the four affected skill/agent/command files in place; do not introduce new skills or split existing ones for these changes.

### Assumptions

- **AS1 — implementer-defer-mode-already-works** — `Commit mode: defer` in `up:implementer` correctly stages files and skips the commit, as documented in `plugins/up/agents/implementer.md`. Verified by reading the file.
- **AS2 — single-phase-detection-is-trivial** — Counting `### PH<N>` headings in `## Plan` reliably identifies single-phase plans; no edge case where a plan has zero or hidden phases.
- **AS3 — hands-off-default-yes-for-commit-prompts-is-safe** — Auto-committing the RFC file in hands-off mode is acceptable because the worktree-first rule prevents the commit from landing on `main`. Verified against `up:handsoff` safety principles.
- **AS4 — chat-to-editor-handoff-is-a-net-win** — Users prefer reviewing long markdown in their editor over inline chat. Stated by the user when requesting change 5.

### Unknowns

- **UK1 — hands-off-single-phase-policy** — Should hands-off ever defer commit for solo phases (e.g. to surface output under `### Deferred`)? Current decision: no, hands-off commits normally. Re-open if a future hands-off run produces output the user would want to inspect.
- **UK2 — design-commit-includes-rename-or-not** — The end-of-`udesign` commit currently stages only the RFC; the `git mv` of the source backlog file (done by `/up:make`) is already staged from earlier. Resolution proposed: include the rename in the same `docs(rfc): design — <slug>` commit. To confirm during planning.

## Plan

Approach: edit six markdown files in place (`udesign/SKILL.md`, `uplan/SKILL.md`, `uexecute/SKILL.md`, `implementer.md`, `implementer-sonnet.md`, `make.md`) plus a patch bump in `plugin.json`. Per the user override, each bullet below carries the proposed verbatim wording — treat the plan as a near-final draft. One file per phase; phases are independent so order is by file size, smallest first.

**Produced-artifacts directive:** Produced artifacts (code, comments, docs) must refer to design entities by their full description. Design IDs (IV*/PC*/AS*/UK*/PH*/RK*/CK*) and short names are task-file-internal shorthands only. Reason: a stranger reading the codebase later has no RFC context to decode either form.

### PH1 — udesign: write-before-review and end-of-stage commit prompt

- **1.1** `plugins/up/skills/udesign/SKILL.md` (modify) — `## Process` block (lines 14-30)

  Replace step 6 (`Present the design in sections. Get per-section approval.`) and steps 9-11 with the new order. Final block reads:

  ```markdown
  1. Explore project context — files, recent commits, existing patterns. No exceptions.
  2. Scope check — split into multiple tasks now if the ask is too large.
  3. Ask clarifying questions, either one at a time or by groups pertaining to a single topic. Where possible, provide options to choose for every question, with a recommended option, based on best practices.
  4. Propose a few reasonable approaches for the task, Each with explicit tradeoffs and unknowns.
  5. Backwards-compat check — flag anything that could break already-running or already-used systems. Ask the user how to resolve before proceeding.
  6. Identify invariants (IV), principles (PC), assumptions (AS), and unknowns (UK).
  7. Decide TDD — yes or no, with reason. Use `up:test-driven-development`'s applicability rule.
  8. Self-review for placeholders, contradictions, scope, ambiguity. Fix inline.
  9. Write the full Design to the task file — `## Design`, `### Invariants`, `### Principles`, `### Assumptions`, `### Unknowns`. Writing precedes review so the user can read it in their editor of choice.
  10. Ask the user to review the written file. Point at the path (`docs/RFCs/<slug>.md`) and request: approve, request changes, or reject. On change requests: edit the file in place and re-prompt at the same path. Do not paste the full Design back into chat — the file is the source of truth.
  11. After approval, propose committing the RFC. Print `Commit RFC with Design? [Y/n]`. On `y`/empty: stage `docs/RFCs/<slug>.md` plus any pre-staged rename of the seeding backlog file (the `git mv` that `/up:make` performed in file mode), and commit with `docs(rfc): design — <slug>`. On `n`: continue without committing. No branch creation at this stage.
  12. Invoke `up:uplan`.
  ```

  - Implements: write-before-review (IV5) by moving the write step before the review step; end-of-stage commit prompt (IV3) by adding step 11; preserves no-worktree-yet (IV2) by explicitly saying "No branch creation at this stage."

- **1.2** `plugins/up/skills/udesign/SKILL.md` (modify) — `## Hands-off mode` section (lines 191-193)

  Append a sentence to the existing paragraph:

  ```markdown
  Hands-off also defaults the end-of-stage commit prompt to `yes`; log `- udesign: RFC auto-committed (hands-off)` under `### Hands-off decisions` and proceed.
  ```

- **1.3** `plugins/up/skills/udesign/SKILL.md` (modify) — `## Terminal state` (lines 195-197)

  Replace the existing one-line terminal state with:

  ```markdown
  ## Terminal state

  Design has been written to `docs/RFCs/<slug>.md`, the user has approved it (interactive) or it has been auto-approved (hands-off), and the end-of-stage commit prompt has been answered (commit landed or user declined). Status → `planning`. Invoke `up:uplan`. Do not write code. Do not invoke any other skill.
  ```

- Respects: "write-before-review" (IV5), "no-auto-commit-without-prompt-in-interactive" (IV3), "worktree-decision-not-before-execute" (IV2)

### PH2 — uplan: write-before-review, symbol-name-required, end-of-stage commit prompt

- **2.1-2.4** dropped
- 
- **2.5** `plugins/up/skills/uplan/SKILL.md` (modify) — `## Process` block (lines 41-53)

  Replace step 11 (`Present the plan to the user. In interactive mode, wait for approval...`) with two steps:

  ```markdown
  1.  Write the full Plan to `docs/RFCs/<slug>.md` (Plan-before-review). Then ask the user to review the written file at that path — approve, request changes, or reject. On change requests, edit in place and re-prompt at the same path.
  2.  After approval, propose committing the RFC. Print `Commit RFC with Plan? [Y/n]`. On `y`/empty: stage `docs/RFCs/<slug>.md` and commit with `docs(rfc): plan — <slug>`. On `n`: continue without committing. In hands-off mode (task-file `**Mode:** hands-off`), default to `yes` and log `- uplan: RFC auto-committed (hands-off)` under `### Hands-off decisions`. Then invoke `up:uexecute`. In hands-off mode, also log `- uplan: plan auto-approved (hands-off)` as today.
  ```

- **2.6** `plugins/up/skills/uplan/SKILL.md` (modify) — `## Hands-off mode` section (lines 194-196)

  Replace the existing paragraph with:

  ```markdown
  See `up:handsoff` for the full contract. Stage-specific delta: skip the approval wait — write the plan to the task file, default the end-of-stage commit prompt to `yes` (logged as `- uplan: RFC auto-committed (hands-off)`), then invoke `up:uexecute` directly. Self-review, scope-creep check, and backwards-compat restatement are unchanged. Ambiguities that would require a user call go to `### Deferred (needs user input)` and the plan stops — do not guess around them.
  ```

- **2.7** `plugins/up/skills/uplan/SKILL.md` (modify) — `## Terminal state` (lines 198-200)

  Replace the existing block with:

  ```markdown
  ## Terminal state

  Plan written to `docs/RFCs/<slug>.md`, self-reviewed, scope-checked. Interactive: user has approved at the file path, end-of-stage commit prompt answered, then `up:uexecute` invoked. Hands-off: plan auto-approved and auto-committed (both logged), then `up:uexecute` invoked.
  ```

- Respects: "symbol-name-required-in-plan-bullets" (IV1), "line-range-is-advisory-symbol-is-authoritative" (PC1), "write-before-review" (IV5), "no-auto-commit-without-prompt-in-interactive" (IV3)

### PH3 — uexecute: worktree-at-start step, solo-phase defer-and-review

- **3.1** `plugins/up/skills/uexecute/SKILL.md` (modify) — `## Before starting` block (lines 10-18)

  Replace the existing `<required>` list with:

  ```markdown
  <required>
  1. Read the full task file. You must work according to the plan provided.
  2. **Worktree / branch decision.** Read the task file's `**Branch:**` header. If it is `main` or `master` and the task is not trivial: stop, prompt the user `Create a dedicated branch + worktree for this task? [Y/n]`. On `y`/empty: invoke `up:git-worktrees`, update the task file's `**Branch:**` and `**Worktree:**` headers, and continue inside the new worktree. On `n`: continue on the current branch and record `- uexecute: user declined worktree, executing on <branch>` under `### Conclusion → ### Deviations from plan` (create the subsection if missing). Hands-off mode: skip the prompt and always create a branch + worktree; the only escape is when `up:git-worktrees` itself fails, in which case log under `### Deferred (needs user input)` and stop. Trivial tasks may stay on the current branch without a prompt.
  3. **Verify working state.** Run independently of whether step 2 created anything: confirm `git rev-parse --show-toplevel` matches the task file's `**Worktree:**` header (or the main repo root when `**Worktree:** none`) and `git branch --show-current` matches `**Branch:**`. Covers the case where step 2 was skipped because the headers already point at a task-specific branch from a previous session. On mismatch: stop and ask.
  4. Build the checklist — one todo per plan phase (or per task if phases are coarse). Use TodoWrite.
  5. **Single-phase detection.** Count `### PH<N>` headings in `## Plan`. If exactly one and no `### Interface graph` section is present, mark the run as **solo-phase**: dispatch with `Commit mode: defer` and follow the solo-phase review protocol in "Dispatch per phase" below.
  </required>
  ```

  - Implements: "worktree-decision-not-before-execute" (IV2) by making uexecute own the decision; "solo-phase-defers-commit-in-interactive" (IV4) by detecting single-phase plans here.

- **3.2** `plugins/up/skills/uexecute/SKILL.md` (modify) — `## Dispatch per phase` block (around line 71-104), specifically the "Pass in the dispatch prompt" `<required>` block (lines 88-104)

  Replace the `Commit mode: self | defer` bullet (currently line 96) with:

  ```markdown
  - `Commit mode: self | defer` — `defer` when the phase is in a multi-phase wave per `### Interface graph`, OR when the run is **solo-phase** in interactive mode (single `### PH<N>` and no graph). Otherwise `self`. In hands-off mode, solo-phase still uses `self` — there is no user to review staged changes.
  ```

- **3.3** `plugins/up/skills/uexecute/SKILL.md` (modify) — append a new subsection after `## Dispatch per phase` and before `## Wave dispatch` (insert around line 124)

  ```markdown
  ## Solo-phase review protocol

  Used when the plan has exactly one `### PH<N>` heading, no `### Interface graph`, and the task-file `**Mode:**` is `interactive`.

  <required>
  1. Dispatch the single implementer with `Commit mode: defer`. The implementer stages files, runs tests, and reports the intended commit message — but does not commit.
  2. On the implementer's `DONE` return, print a short summary to the user:
     - `git diff --stat --staged` output
     - The implementer's proposed commit message
     - The implementer's `Implemented:` and `Tests:` / `Smoke:` lines verbatim
  3. Ask: `Now, you can review the staged changes. Proposed commit message: <...>. Do you approve to commit? [y/n/other]`.
     - `y`  → check every touched file was staged (user may have changed smth: check that no file appears in both staged and changed, not staged sets). Run `git commit -m "<proposed message>"`, then proceed to the plan-diff check, consistency pass, and `up:uverify` as in the normal flow.
     - `n` → do not commit. Log `- uexecute: user declined commit of staged solo-phase changes` under `### Conclusion → ### Deferred (needs user input)` (create the subsection if missing), with the proposed message and staged file list. Stop. Do not invoke `up:uverify`.
     - `other` -> allow user free-text answer.
  4. On `DONE_WITH_CONCERNS`: print the concerns alongside the diff summary; otherwise behave as `DONE`.
  5. On `BLOCKED` / `NEEDS_CONTEXT`: handle as in the normal serial-fallback loop — re-dispatch with corrected context or escalate to `up:uplan`.
  </required>

  Hands-off mode: skip this protocol entirely. Solo-phase dispatches use `Commit mode: self` and follow the normal serial-fallback loop. Log `- uexecute: solo-phase auto-committed (hands-off)` under `### Hands-off decisions`.
  ```

  - Implements: "solo-phase-defers-commit-in-interactive" (IV4).

- **3.4** `plugins/up/skills/uexecute/SKILL.md` (modify) — `## Hands-off mode` block (lines 311-315)

  Append a sentence:

  ```markdown
  Solo-phase behavior in hands-off is **not** deferred — the implementer is dispatched with `Commit mode: self` because there is no user to review the staged changes (UK1 in the originating task; see "Solo-phase review protocol" above).
  ```

- Respects: "worktree-decision-not-before-execute" (IV2), "solo-phase-defers-commit-in-interactive" (IV4), "implementer-defer-mode-already-works" (AS1), "single-phase-detection-is-trivial" (AS2)

### PH4 — /up:make: drop the pre-execute worktree step

This phase removes the pre-execute worktree decision from `/up:make` (its new home is PH3's `up:uexecute` edit). PC1 (line-range-advisory) is enforced without any dedicated implementer-agent edit: the existing `## When to stop and ask` rule in `up:uexecute` ("A dependency the plan assumes is missing") and the generic implementer rule ("If anything critical is missing or ambiguous, stop and ask before writing code") already require the implementer to halt on a missing or renamed symbol. The directive in `uplan/SKILL.md` ("Note that line numbers are subject to change, use the symbolic names … as the ultimate source of truth. Raise error if those are missing.") makes the planner-side contract explicit.

- **4.1** `plugins/up/commands/make.md` (modify) — `### 6. Branch & worktree decision` section (lines 116-127)

  Delete the entire section. Renumber the subsequent sections so the flow becomes 1-Slug, 2-Resume, 3-Create-task-file, 4-Size, 5-Design, 6-Plan (formerly 7), 7-Execute (formerly 8), 8-Verify (formerly 9), 9-Review (formerly 10), 10-Finish (formerly 11). Update all internal references in `make.md` that name a step number.

  In place of the deleted section, insert one paragraph that signposts the move:

  ```markdown
  ### 6. Plan stage (unless skipped)

  Invoke `up:uplan`. It populates `## Plan`. After plan approval (and the end-of-stage commit prompt the skill now owns), Status → `executing`.

  Worktree and branch creation is no longer decided here. `up:uexecute` owns that step — it prompts at the start of execution (interactive) or auto-creates a worktree (hands-off). See `up:uexecute → ## Before starting` step 2.
  ```

- **4.2** `plugins/up/commands/make.md` (modify) — `## Hands-off mode` block (lines 200-202)

  Replace the reference to "step 4, step 6, step 7, step 11" with the new numbering:

  ```markdown
  Activated by prefixing `/up:make` arguments with the literal token `handsoff`. The full contract — safety principles (worktree-first, reversible-first, no destructive ops, no push), decision log format, deferred log, no-default rule, end-of-task summary — lives in `up:handsoff`. Read that skill once when the task file's `**Mode:**` header is `hands-off`.
  ```

- **4.3** `plugins/up/commands/make.md` (modify) — `## Rules` list (lines 189-198)

  Drop the rule `Never create a worktree without confirming in interactive mode`. No replacement — the rule moves into `up:uexecute → ## Before starting` step 2.

- Implements: "worktree-decision-not-before-execute" (IV2) by removing the only place /up:make would create a worktree before execute.

- Respects: "worktree-decision-not-before-execute" (IV2), "additive-skill-changes-only" (PC2) — this is the one subtractive edit, justified because the step has a new home.

### PH5 — version bump

- **5.1** `plugins/up/.claude-plugin/plugin.json` (modify) — version field

  ```diff
  -  "version": "0.3.9",
  +  "version": "0.3.10",
  ```

### Test strategy

No automated tests (markdown-only project, per CLAUDE.md "no runtime code, no unit tests; verification is install-and-invoke"). Verification belongs to `up:uverify` and runs as:

- **Manual smoke 1 — udesign write-before-review:** start a tiny new task via `/up:make handsoff "rename foo to bar"`; during design, confirm the Design section appears in `docs/RFCs/rename_foo_bar.md` *before* any approval prompt is issued.
- **Manual smoke 2 — udesign commit prompt:** in interactive mode, confirm the `Commit RFC with Design? [Y/n]` prompt appears after approval, and on `y` a `docs(rfc): design — <slug>` commit lands containing the RFC file and any pre-staged backlog rename.
- **Manual smoke 3 — worktree at uexecute:** with task `**Branch:** main`, invoke uexecute; confirm the worktree prompt fires before any read of the plan body.
- **Manual smoke 4 — solo-phase defer:** plan a single-phase task; confirm the implementer stages but does not commit; confirm uexecute prints `git diff --stat --staged` and the `Now, you can review the staged changes...` prompt; on `y`, confirm one commit lands with the proposed message.
- **Manual smoke 5 — hands-off solo-phase:** same as 4 but with `handsoff` keyword; confirm `Commit mode: self` (no defer) and that the implementer commits directly without an approval prompt.
- **Manual smoke 6 — version visible:** `/reload-plugin` and confirm `0.3.10` is the loaded version.

User performs further validation outside of this list.

### Order & dependencies

PH1 → PH2 → PH3 → PH4 → PH5 strictly serial; each phase edits a single file (PH2 and PH3 each touch one SKILL.md; PH4 touches `make.md`; PH5 touches `plugin.json`). No cross-phase symbol coupling. Per user direction, all five phases land in **one combined commit** at the end of execution (not one per phase).

Mechanism: dispatcher dispatches each of PH1-PH5 with `Commit mode: defer`, accumulates staged changes across phases, then runs a single final `git commit -m "feat: stable code references and lifecycle commit prompts"` after PH5's self-review passes.

### Risks / rollback

- RK1 — A user resuming `docs/RFCs/no_shorthand_references.md` (the one in-flight task, status `planning`) experiences the new `up:uexecute → ## Before starting` worktree prompt on a Plan written under the old format. Mitigation: the new step is purely additive — it asks before reading the plan body and never rewrites it. Rollback: revert PH3 if reports of resume breakage appear.
- RK2 — Solo-phase defer protocol introduces a new interactive prompt; if a user runs `up:uexecute` in a non-interactive harness (script, CI), the prompt blocks indefinitely. Mitigation: the protocol is interactive-only by design and is gated by `**Mode:** interactive`; hands-off bypasses it (PH3.4 makes this explicit).

### Self-review

1. Spec coverage —
   - IV1 (symbol-name-required) — uplan/SKILL.md line 59 already mandates "+ affected class/method names (mandatory)"; the in-place edit at line 70 adds the explicit "Raise error if those are missing" directive. No plan phase needed.
   - IV2 (worktree-not-before-execute) — PH3.1 (uexecute adds the prompt), PH4.1 (make.md removes the early step).
   - IV3 (no-auto-commit-without-prompt) — PH1.1 step 11 (udesign), PH2.5 step 12 (uplan).
   - IV4 (solo-phase-defers-commit) — PH3.1 step 5 (detection), PH3.2 (dispatch wiring), PH3.3 (protocol).
   - IV5 (write-before-review) — PH1.1 steps 9 and 10 (udesign), PH2.5 step 11 (uplan).
   - PC1 (line-range-advisory) — enforced by the symbolic-names directive in `uplan/SKILL.md` plus the existing generic stop-and-ask rules in `up:uexecute` and `implementer.md`; see PH4 intro paragraph.
   - PC2 (additive-skill-changes-only) — PH4 is the one subtractive edit, justified because the deleted step has a new home in PH3.
   - AS1 (defer-mode-works) — PH3 relies on existing implementer.md `Commit mode: defer` behavior; not edited.
   - AS2 (single-phase-detection-trivial) — PH3.1 step 5 counts `### PH<N>` headings.
   - AS3 (hands-off-default-yes-safe) — PH3.4 makes hands-off solo-phase explicit (use `self`).
   - AS4 (chat-to-editor-handoff) — motivates PH1.1 and PH2.5.
   - UK1 (hands-off-single-phase-policy) — resolved by PH3.4: hands-off uses `self`, not defer.
   - UK2 (design-commit-includes-rename) — resolved by PH1.1 step 11: the rename is included in the same `docs(rfc): design — <slug>` commit.
2. Placeholder scan — no "TBD" / "handle edge cases" / "similar to" / "write tests for the above". ✓
3. Consistency — `Commit mode: self | defer` wording matches existing uexecute SKILL.md; commit message conventions match recent git log (`feat(scope):`, `refactor(scope):`, `chore:`). ✓
4. Leanness — five phases, single combined commit; matches a markdown-only task expected to take one session. ✓
5. IV / AS coverage — each IV named by ID in the spec-coverage list above. ✓
6. Interface graph — omitted: every phase edits a different file with no cross-phase contract. ✓

### Scope-creep / simpler-way check

The phases each map one-to-one to a Design change; no "while I'm here" refactors crept in. A simpler alternative (one mega-edit with no phase structure) would lose the per-file review boundary; the chosen shape keeps reviewability while still landing as a single commit per user direction.

### Backwards-compat (restated from Design)

- In-flight task `no_shorthand_references.md` (status `planning`): on resume, PH3.1 adds a worktree prompt before any plan read. Additive only — no rewrite of the existing Plan section.
- Closed task RFCs: never re-read by these skills; no impact.
- `Commit mode: defer` existing wave-dispatch behavior: PH3.2 widens the trigger condition (adds solo-phase to the set), does not change the protocol the implementer follows.

## Verify
<empty — filled by up:uverify>

## Conclusion
<empty — filled by up:ureview>

### Hands-off decisions
<empty — populated only when Mode is hands-off>

### Deferred (needs user input)
<empty — populated only when Mode is hands-off and a choice had no conservative default>
