---
description: Orchestrate the full ultrapack workflow — slug, task file, design, plan, execute, verify, review. Size-aware, resume-ready. Prefix args with `handsoff` for hands-off mode (fewer prompts, conservative defaults, decision log).
---

# /up:make

Drives a task through the full ultrapack workflow: one task file at `docs/RFCs/<slug>.md`, evolving through Design → Plan → Conclusion. Each stage is a separate skill. You orchestrate; the skills do the work.

## Arguments

The user's description of the task follows the command. May be a one-liner ("fix the flaky login test") or a paragraph. Use it as the seed for the slug and the initial framing for `up:udesign`.

Three accepted intake shapes: **text-only** (current behavior — args are pure description), **file-only** (args are a single reference to an existing `.md` file), **file + text** (args contain one file reference plus additional descriptive text in any order).

Parsing rule: whitespace-split the args; strip an optional leading `@` from each token; classify each token as a file reference if the stripped token resolves to an existing regular `.md` file (if it lacks folder name: glob it, to verify file existence). 

Hands-off activation: if the first whitespace-delimited token of the arguments is the literal string `handsoff`, enable hands-off mode. Strip that token before deriving the slug or framing for design. Any other spelling (`hands-off`, `handsOff`, `--handsoff`) is treated as part of the description — only the bare token `handsoff` activates. See `## Hands-off mode` below for behavior. The `handsoff` token is stripped before the file-detection parsing rule runs.

## Flow

### 1. Slug

Text-only mode: derive a snake_case slug from the description, 3 words max (e.g. "flaky_login_test"), and proceed. The slug is snake_case, as is any doc file this workflow creates.

File mode (file-only or file + text): the slug equals the input file's basename without its `.md` extension, used verbatim, with no length limit and no word-count cap. The basename (sans extension) must match `^[a-z0-9]+(_[a-z0-9]+)*$`; if it does not, `/up:make` rejects the input with a user-visible message naming the offending basename and stops. No silent normalization. No silent fallback to text-only mode.

### 2. Resume check

Before creating a new task file, check if `docs/RFCs/<slug>.md` already exists.

- Exists: read `**Status:**` from the header. Resume from the next stage:
  - `design` → continue design
  - `planning` → run `up:uplan`
  - `executing` → run `up:uexecute`
  - `reviewing` → run `up:ureview`
  - `done` → ask the user what they want to do (start a follow-up, re-open, view conclusion)
- Doesn't exist: proceed to step 3.
- Multiple in-flight tasks: if more than one `docs/RFCs/*.md` has Status ≠ `done`, list them and ask which one the user means (or whether this is a new task).

### 3. Create task file

Create `docs/RFCs/<slug>.md` from the template. Status = `design`. Branch = `main` (placeholder; set later by `up:uexecute` when it creates a branch + worktree). No worktree. Mode = `hands-off` if the keyword was present, else `interactive`.

Seed `## Original description` based on intake shape:

- **Text-only mode:** insert the user's text verbatim.
- **File-only mode:** insert the input file's body, with every embedded markdown heading demoted by exactly one level (`#`→`##`, `##`→`###`, `###`→`####`, …). This guarantees no H2 appears inside the section.
- **File + text mode:** insert the user's text as a leading paragraph, then a blank line, then the demoted file body (same demotion rule as file-only mode).
- **Already-wrapped escape hatch:** if the input file already contains an `## Original description` section, copy that section's body verbatim into the new task file (no demotion, no wrapping) — this prevents double-wrapping when re-running `/up:make` on a file that previously came out of this workflow.

Filesystem effects in file mode:

- If the input file is **outside** `docs/RFCs/`: after the new RFC at `docs/RFCs/<slug>.md` is written, run `git mv <input-path> <input-dir>/wip_<slug>.md` as part of the same `/up:make` invocation. If `<input-dir>/wip_<slug>.md` already exists, refuse and stop with a user-visible message — do not overwrite, do not silently pick another name.
- If the input file is **inside** `docs/RFCs/`: skip the wrap-and-rename. The Resume check in step 2 owns this case (the file is already a task file, possibly mid-flight).

Template:

```markdown
# <Task Title>

**Status:** design
**Branch:** main
**Worktree:** none
**Mode:** <interactive|hands-off>

## Original description
<empty — filled by /up:make from input>

## Design
<empty — filled by up:udesign>

### Invariants
<empty — IV1, IV2, … : hard constraints that must hold>

### Principles
<empty — PC1, PC2, … : softer guidance>

### Assumptions
<empty — AS1, AS2, … : unverified premises the design rests on; conclusion must report whether each held>

### Unknowns
<empty — UK1, UK2, … : open questions left to plan / execute; conclusion must report whether each resolved>

## Plan
<empty — filled by up:uplan>

## Verify
<empty — filled by up:uverify>

## Conclusion
<empty — filled by up:ureview>

### Hands-off decisions
<empty — populated only when Mode is hands-off>

### Deferred (needs user input)
<empty — populated only when Mode is hands-off and a choice had no conservative default>
```

### 4. Size classification

Based on the task description, classify size:

- Trivial — one-line change, typo, rename. Skip Design and Plan. Go straight to Execute. Status file still created.
- Small — single file or single concept change. Skip Design. Plan runs.
- Medium / Large — full flow.

Interactive mode: default to Medium silently. Jump to Trivial/Small only when the user's wording signals it — e.g. "quickly", "fast", "just", "one-line", "typo", "rename". Confirm before skipping any stage. When genuinely ambiguous, ask.

Hands-off mode: do not confirm. Default to Medium (full flow) unless the scope is unambiguously Trivial (true one-liner in one file). Never auto-pick Small or auto-skip Design — Design is the one interactive stage preserved in hands-off. Append the choice to `## Conclusion → ### Hands-off decisions` as `- size: <classification> — <rationale>`.

### 5. Design stage (unless skipped)

Invoke `up:udesign`. It populates `## Design`, `### Invariants` (IV), `### Principles` (PC), `### Assumptions` (AS), `### Unknowns` (UK), and records `TDD: yes / no (reason)`. Status → `planning`.

### 6. Plan stage (unless skipped)

Invoke `up:uplan`. It populates `## Plan`. After plan approval (and the end-of-stage commit prompt the skill now owns), Status → `executing`.

Worktree and branch creation is no longer decided here. `up:uexecute` owns that step — it prompts at the start of execution (interactive) or auto-creates a worktree (hands-off). See `up:uexecute → ## Before starting` step 2.

### 7. Execute stage

Invoke `up:uexecute`. Implements the plan, commits incrementally.

### 8. Verify loop

Invoke `up:uverify`. On failure: `up:uverify` describes how each failure *should* have worked, control returns to `up:uexecute`. Loop until verify passes.

### 9. Review stage

Status → `reviewing`. Invoke `up:ureview`. It dispatches `up:ureviewer`, processes findings, fills `## Conclusion`. Status → `done`.

Once the task is concluded as `done`, run the docs-refresh check (see below).

### 10. Finish

Hands-off mode — first: print the `## Conclusion → ### Hands-off decisions` list (and `### Deferred (needs user input)` if non-empty) to the user and ask verbatim: "Here's what I did to make it hands-off. Want to change anything?" Wait for the user's response before continuing.

Then (both modes) present options to the user:
- Merge / open PR (if on a branch)
- Clean up worktree
- Move on

Execute only after the user chooses.

## After task is done — docs refresh

Run this once, after Review concludes the task as `done` (not after every stage). Scan the project docs and update them if the work surfaced something they should reflect. Cheap, light-touch; not a full doc pass.

Files to scan:
- `CLAUDE.md` (project-wide agent guidance)
- `README.md`
- `docs/**/*.md` (project documentation, excluding the task file itself and archived tasks)

What to look for:
- New conventions, invariants, or principles that should be global → update `CLAUDE.md`
- New components, commands, or features the README should mention
- Stale content contradicted by the stage's work → delete or correct
- Pointers to the task file if future contributors would benefit

Rules:
- If nothing needs updating: say so in one line and move on. Do not invent edits.
- If updates are needed: make them directly, then summarize what changed in 1-3 lines (e.g. "README: fixed install instructions; CLAUDE.md: no change"). Do not prompt for approval first. Do not produce a detailed diff — the user will git-diff if they want.
- Follow `up:udocument`: lead with why, lists over tables, no aspirational content, kill stale content.
- Do not duplicate content across task file and project docs — pick one home per fact.

## Stop conditions

Stop and ask the user when:

- Size classification is genuinely unclear (interactive only; in hands-off, default to Medium)
- User has expressed a preference (branch, scope, TDD) that conflicts with the auto-inference
- Any stage's skill returns a blocker

## Rules

- Never skip Review (both modes)
- Never auto-merge or auto-push — the user chooses at step 10 (both modes)
- Never edit `main` / `master` directly in hands-off (see `up:handsoff` safety principles)
- Keep the task file as the single source of truth — each stage reads it, each stage writes to it
- External spec / design docs (e.g. anything under `docs/specs/`) are read-only during execute. If a stage finds the spec is wrong, surface it to the user — don't mutate it silently
- Don't assume prior session memory — the next agent may be a fresh context reading only the task file
- In hands-off, never invent a default for an ambiguous argument — see `up:handsoff` no-default rule

## Hands-off mode

Activated by prefixing `/up:make` arguments with the literal token `handsoff`. The full contract — safety principles (worktree-first, reversible-first, no destructive ops, no push), decision log format, deferred log, no-default rule, end-of-task summary — lives in `up:handsoff`. Read that skill once when the task file's `**Mode:**` header is `hands-off`.

## Terminal state

Task file Status = `done`, Conclusion filled, user has chosen a finish action (merge, PR, cleanup, or defer). In hands-off, the user has also reviewed the `### Hands-off decisions` list.
