# Free-form File Input to `/up:make`

**Status:** planning
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
<empty — filled by up:uplan>

## Verify
<empty — filled by up:uverify>

## Conclusion
<empty — filled by up:ureview>
