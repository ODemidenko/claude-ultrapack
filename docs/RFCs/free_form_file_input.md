# Free-form File Input to `/up:make`

**Status:** reviewing
**Branch:** task/free_form_file_input
**Worktree:** .worktrees/task/free_form_file_input
**Mode:** interactive

## Original description

There may be a case when user provides a file as input to the whole "ultrapack" system,
this is usually a vague feature draft.

update the `make.md` command description and, if required all the skills it references, so that they can accommodate scenario when `make` command or udesign skill are called with a file reference.

Scenarios to be supported:
- `make.md` is called with a file name that is out of "RFCs" folder. *Example of such a file is this one.*
    - Then, reuse the same filename for a new file, created in the `RFCs` folder. This name must be the "slug"
    - And, to distinguish filenames, the original filename must be prefixed with `wip_`
- `make.md` command is called with a file that is already in `RFCs` folder. In this case, `make` command edits this file in place.

In both cases:
file content is regarded as a first draft, and its original wording, must be put, with minimal reasonable changes (grammar fixes / converting all headers to start from H2), into the "## Original description" in the RFC description that `make.md` command creates.

### Design step
  - `design/SKILL.md` must expect that input file *may, but is not required* to contain the "## Original description" heading
  - design skill may and is expected to change the original description, in case any clarifications come up during the design process
  Though, in case there were no major change in direction, it should rather keep the original structure/wording, as the user would prefer to recognize the initial information exactly in his own words. And there is a further "## Design" section with a goal to disambiguate.
  - under the "## design" part that the design skill is filling, there can be a reasonable duplication/restructuring of the information, provided in the "

### Future steps
Check if any other files have rigid reference to "## Design" part in the RFC doc, which may lead to them omitting the original description.
Wherever the "## Design" part is visible, the "## Original description" part must be visible as well.

## Design

Extend `/up:make` to accept any combination of (markdown file path) and
(text description) as its argument, and to seed the resulting RFC with
a preserved-verbatim user-input section. Scope: doc-only edits to a
small set of plugin files (`/up:make`, `udesign`, `uplan`, `uexecute`,
`ureview`, `agents/reviewer.md`). Out of scope: non-markdown file
inputs, multi-file inputs, retroactive migration of existing RFCs.

**Postel's-law parsing.** `/up:make` is liberal in what it accepts and
strict in what it returns. Args are whitespace-split into tokens; each
token is classified by trying to resolve it as a file path:

- After stripping an optional leading `@`, if the token resolves to an
  existing regular file with a `.md` extension, it is a **file
  reference**.
- Remaining tokens form the **text portion**.

The first matching file token wins; subsequent file-shaped tokens that
also resolve cause `/up:make` to stop and ask. A bare invocation with
no args also stops and asks.

**Three modes** result from this parsing:
- text only — current behavior, preserved
- file only — new
- file + text — new

**Slug derivation.**
- Text-only mode: existing rule unchanged — snake_case, ≤3 words.
- File mode (with or without text): slug equals the file basename
  without extension, verbatim. No length limit. No rewriting. The
  basename must match `^[a-z0-9]+(_[a-z0-9]+)*$`; otherwise `/up:make`
  rejects with a user-visible message — no silent normalization, no
  silent fallback to text mode.

**Strict output: `## Original description` always present.** Every
RFC `/up:make` produces from now on contains a `## Original
description` section, placed immediately before `## Design`. Section
content depends on mode:

- text only — the user's text, verbatim
- file only — the file's content, with every embedded markdown
  heading demoted by exactly one level (`#`→`##`, `##`→`###`, …). If
  the file already contains `## Original description`, copy that
  section's body verbatim instead of wrapping (no double-wrap).
- file + text — the user's text as a leading paragraph, blank line,
  then the file content (demoted as above). Same double-wrap
  avoidance applies if the file already has its own `## Original
  description`.

**Filesystem effects (file mode, file outside `docs/RFCs/`).** After
writing the new RFC, `git mv` the original file from `<dir>/<slug>.md`
to `<dir>/wip_<slug>.md`. Refuse and surface to the user if
`<dir>/wip_<slug>.md` already exists at the destination — never
overwrite.

**Filesystem effects (file mode, file inside `docs/RFCs/`).** Skip the
wrap and rename. The existing Resume flow takes over because the slug
already matches an RFC file. The `## Original description` section,
if any, is preserved by virtue of not being touched.

**Sibling-skill awareness.** Five plugin files instruct an agent to
read `## Design` (or its child sections) from a task file. Each is
updated to also read `## Original description` if present:

- `plugins/up/agents/reviewer.md`
- `plugins/up/skills/udesign/SKILL.md`
- `plugins/up/skills/uplan/SKILL.md`
- `plugins/up/skills/uexecute/SKILL.md`
- `plugins/up/skills/ureview/SKILL.md`

One-line additive edit per file. No removals.

**udesign behavior.** Treat `## Original description` as the user's
authentic initial framing. Default: preserve verbatim. udesign may
edit only when a design clarification *materially* contradicts or
sharpens the original phrasing, and even then prefers minimal in-place
edits over rewrites. Stylistic cleanup is not allowed — the user
wants to recognize their own words on subsequent reads. udesign also
tolerates the absence of `## Original description` (older RFCs, or
text-only inputs whose Original description happens to be empty).

**ureview / reviewer behavior.** The dispatched `up:reviewer` reads
both `## Design` and `## Original description` from the task file.
The reviewer compares the delivered work against both: Design (the
team's promises) and Original description (the user's original ask).
Drift between the two — or between the delivered work and either —
is a finding to surface in `## Conclusion`.

**Decisions and tradeoffs.**
- *Hard-reject non-snake_case filenames* (vs. silent normalize):
  preserves round-trip identity between input file, slug, and
  `wip_`-prefixed file. Cost: small friction when the user picks a
  wrong-shaped name. Worth it — silent rewriting would yield a
  `wip_` file whose name no longer matches the RFC.
- *Demote-by-one for embedded headings* (vs. literal "min H2"):
  preserves nesting under `## Original description`. Costs one regex
  pass at wrap time; gains parser-safe section boundaries.
- *`## Original description` always present in output* (vs.
  conditional): uniform RFC shape simplifies downstream skills'
  reading logic — they always know where to look. Cost: text-only
  mode now seeds the section with the user's args (previously the
  args lived only as conversation context).
- *Concentrate logic in `/up:make`* (vs. new `up:ufile-input` skill):
  `/up:make` is already the orchestrator that owns slug derivation
  and file creation. New skill would violate CLAUDE.md "Minimal — no
  speculative additions".

TDD: no (doc-only change to plugin skill files; verification is
install-and-invoke per repo CLAUDE.md design principle).

### Invariants

- IV1 — In file mode, the slug equals the input file's basename
  without extension. The new RFC file is `docs/RFCs/<slug>.md`.
- IV2 — When the input file is outside `docs/RFCs/`, the original
  is renamed (via `git mv`) to `<original-dir>/wip_<slug>.md` as part
  of the same `/up:make` invocation that creates the new RFC. No
  partial state in which both the original and the RFC exist with
  the same basename.
- IV3 — Every plugin file under `plugins/up/` that mentions reading
  or writing `## Design` from a task file also mentions
  `Original description`. Verifiable:
  `git grep -lF '## Design' plugins/up/ | xargs -r grep -L 'Original description'`
  returns no output.
- IV4 — Embedded markdown headings inside an `## Original
  description` section produced by `/up:make`'s wrap step are at
  level H3 or deeper. No H2 appears inside the section.
- IV5 — `/up:make` rejects, with a user-visible message, an input
  file whose basename does not match `^[a-z0-9]+(_[a-z0-9]+)*$`. No
  silent normalization. No silent fallback to text mode.
- IV6 — Every RFC created by `/up:make` from feature-completion
  onward contains an `## Original description` section before
  `## Design`. Existing pre-feature RFCs are not migrated; the
  feature is not retroactive.

### Principles

- PC1 — Preserve the user's wording in `## Original description`.
  udesign must not rewrite for stylistic or organizational reasons;
  edit only when a design clarification materially supersedes the
  original phrasing, and prefer minimal in-place edits over rewrites.
- PC2 — Review against the original ask, not just the design. The
  `up:reviewer` agent compares delivered work against both
  `## Design` and `## Original description`; drift between them — or
  between the delivered work and either — is a finding for the
  Conclusion.

### Assumptions

- AS1 — The five files surfaced by the IV3 grep
  (`agents/reviewer.md`, `udesign/SKILL.md`, `uplan/SKILL.md`,
  `uexecute/SKILL.md`, `ureview/SKILL.md`) are the complete set
  of plugin files that process Design content. No skill computes
  the heading dynamically. Conclusion must confirm via a fresh
  `git grep -lF '## Design' plugins/up/` returning the same set.
- AS2 — Claude Code's `@<path>` chat-input shorthand delivers the
  literal string `@<path>` as the slash-command's argument
  (verified in this session: args were
  `@docs/backlog/free_form_file_input.md`). The optional `@`-strip
  in the parsing rule also handles users who type the bare path
  without `@`. If a future Claude Code version pre-resolves `@` to
  inline file content, the parsing rule still works because the
  fallback is to treat the args as text.
- AS3 — `git mv` to `<original-dir>/wip_<slug>.md` preserves git
  history (including any prior commits that touched the original
  file). Standard git behavior; no further verification needed.

## Plan

Approach: two-phase doc edit. PH1 lands the feature in `/up:make`
(file detection, slug rule, RFC template, wrap & rename). PH2 sweeps
five sibling skill/agent files to read `## Original description`
alongside `## Design`. PH2 consumes the section-name contract IF1
that PH1 produces.

### PH1 — `/up:make` file-input mode

- **1.1** `plugins/up/commands/make.md:9-13` (modify) — `## Arguments`
  block. Spell out three accepted intake shapes: text-only (current),
  file-only, file + text. State the parsing rule in one sentence:
  whitespace-split the args; strip an optional leading `@`; classify
  each token as a file reference iff it resolves to an existing
  regular `.md` file. First file token wins; subsequent file-shaped
  tokens that also resolve cause `/up:make` to stop and ask. Bare
  invocation with no args also stops and asks.
  - Respects: AS2
- **1.2** `plugins/up/commands/make.md:17-19` (modify) — `### 1. Slug`
  block. Keep the existing snake_case + 3-words rule for text-only
  mode. Append: in file mode (with or without text), slug = input
  file's basename without extension, verbatim, no length limit;
  basename must match `^[a-z0-9]+(_[a-z0-9]+)*$` or `/up:make`
  rejects with a user-visible message — no silent normalization, no
  fallback to text mode.
  - Respects: IV1, IV5
- **1.3** `plugins/up/commands/make.md:34-37` (modify) — extend the
  `### 3. Create task file` preamble to specify content seeding for
  `## Original description`: text-only mode → user's text verbatim;
  file-only mode → file body with every embedded markdown heading
  demoted by exactly one level (`#`→`##`, `##`→`###`, …); file +
  text mode → text as a leading paragraph, blank line, then the
  demoted file body. If the input file already contains
  `## Original description`, copy that section's body verbatim
  instead of wrapping (no double-wrap).
  - Respects: IV4, PC1
- **1.4** `plugins/up/commands/make.md:34-37` (modify, same step) —
  add filesystem effects: when the input file is outside
  `docs/RFCs/`, `git mv` it to `<dir>/wip_<slug>.md` as part of the
  same `/up:make` invocation; refuse if the destination already
  exists. When the input file is inside `docs/RFCs/`, skip wrap
  and rename — let the existing Resume check (step 2) take over.
  - Respects: IV2
- **1.5** `plugins/up/commands/make.md:38-77` (modify) — RFC
  template block. Insert
  `## Original description\n<empty — filled by /up:make from input>`
  immediately before `## Design`.
  - Respects: IV6
- Commit: `make: file-input mode for /up:make (text/file/file+text)`

### PH2 — Sibling-skill awareness sweep

- **2.1** `plugins/up/agents/reviewer.md:15` (modify) — extend the
  reading-list sentence so the reviewer also reads
  `## Original description` if present. In `### 1. Plan alignment`
  (around line 21-28), add one sentence capturing PC2: compare the
  diff against both the Plan's promises and the user's original ask;
  drift between them is a Plan finding.
  - Respects: IV3, PC2
- **2.2** `plugins/up/skills/udesign/SKILL.md:23-27, 181-188`
  (modify) — in the `## Process` block (step 9 area), add a bullet:
  udesign reads `## Original description` if present. In the
  `## Rules` block (around line 181-188), add one rule capturing
  PC1: preserve `## Original description` verbatim by default; edit
  only when a design clarification materially supersedes the
  original phrasing; prefer minimal in-place edits; no stylistic
  rewrites.
  - Respects: IV3, PC1
- **2.3** `plugins/up/skills/uplan/SKILL.md:42` (modify) — extend
  the reading list to include `## Original description` if present.
  - Respects: IV3
- **2.4** `plugins/up/skills/uexecute/SKILL.md:88-103, 105-114`
  (modify) — extend the "Pass in the dispatch prompt" required list
  to include `## Original description` if present in the task file.
  Update the dispatch prompt skeleton block to include an
  `Original description` field.
  - Respects: IV3
- **2.5** `plugins/up/skills/ureview/SKILL.md:23-33, 45-61`
  (modify) — in the `<reviewer-role>` block, add one sentence
  capturing PC2: critical attention to drift between delivered work
  and the user's original ask. In `### 1. Dispatch up:reviewer`,
  add `## Original description` (if present) to the dispatch
  prompt's skeleton and to the agent's reading list.
  - Respects: IV3, PC2
- Commit: `skills: read ## Original description alongside ## Design`

### Risks / rollback

- RK1 — Wording drift: a sibling-skill edit might unintentionally
  alter behavior beyond the additive intent. Mitigation: every PH2
  edit is strictly additive (no removals, no rephrasings of existing
  sentences); IV3's grep formula
  (`git grep -lF '## Design' plugins/up/ | xargs -r grep -L 'Original description'`)
  verifies coverage at uverify time.
- Rollback: each phase is a single commit on the task branch;
  reverting PH2 restores prior sibling-skill behavior without
  touching `/up:make`; reverting PH1 restores prior `/up:make`
  without touching siblings. Worktree-isolated.

### Interfaces

- IF1 — `## Original description` task-file section. PH1 mandates it
  in `/up:make`'s template, parsing, and wrap rules. PH2 instructs
  five sibling skill/agent files to read it (if present) alongside
  `## Design`.

### Interface graph

- PH1                   -> IF1   @ plugins/up/commands/make.md
- PH2  IF1              ->       @ plugins/up/agents/reviewer.md, plugins/up/skills/udesign/SKILL.md, plugins/up/skills/uplan/SKILL.md, plugins/up/skills/uexecute/SKILL.md, plugins/up/skills/ureview/SKILL.md

## Verify

**Result:** passed

Positive:
- CK1 — RFC template now contains `## Original description` immediately before `## Design` (`make.md:65-67`)
- CK2 — Arguments block names the three intake shapes and the Postel parsing rule (`make.md:13-17`)
- CK3 — Slug rule for file mode + snake_case regex `^[a-z0-9]+(_[a-z0-9]+)*$` documented (`make.md:25`)
- CK4 — Step 3 documents content-seeding rules and `git mv` to `wip_<slug>.md` (`make.md:44-55`)
- CK5 — All five sibling files now mention `## Original description` in their reading list
- CK6 — udesign PC1 rule (preserve verbatim) at `udesign/SKILL.md:189`
- CK7 — PC2 sentence at `agents/reviewer.md:28` and `ureview/SKILL.md:26`

Negative:
- CK8 — PH2 diff is purely additive — `git diff main..HEAD plugins/up/{agents,skills} --shortstat` shows 12 insertions / 4 deletions, the 4 deletions being one-line replacements that retain all original wording
- CK9 — Branch diff touches only the seven expected files (6 plugin files + task file)

Invariants / assumptions:
- CK10 (IV1) — slug = file basename verbatim, documented in `make.md:25`
- CK11 (IV2) — `git mv` to `wip_<slug>.md` in the same `/up:make` invocation, documented in `make.md:53`
- CK12 (IV3) — `git grep -lF '## Design' plugins/up/ | xargs -r grep -L 'Original description'` → empty
- CK13 (IV4) — embedded headings demoted by exactly one level in the wrap step (`make.md:46-49`)
- CK14 (IV5) — non-snake_case basename triggers a user-visible rejection (`make.md:25`)
- CK15 (IV6) — RFC template carries `## Original description` before `## Design` (`make.md:65-67`)
- CK16 (AS1) — five-file completeness implied by CK12; any other plugin file reading `## Design` would fail the IV3 grep
- CK17 (AS2) — unverifiable in this layer; the `@`-strip degrades gracefully if Claude Code ever pre-resolves `@` to inline content
- CK18 (AS3) — `git mv` history-preservation is standard git behavior; precedent shipped in `slugs_and_rfcs.md`

Interfaces:
- CK19 (IF1) — `## Original description` section. Producer: `make.md` template + seeding rules. Consumer: five sibling files. End-to-end wiring verified by CK12.

Smoke: this very task file is a self-demonstration — slug `free_form_file_input` equals the input file's basename, `## Original description` is present, the source was renamed to `docs/backlog/wip_free_form_file_input.md`. End-to-end automated smoke requires the user to re-run `/plugin install up@ultrapack` and invoke `/up:make` against a fresh sample file.

## Conclusion
<empty — filled by up:ureview>
