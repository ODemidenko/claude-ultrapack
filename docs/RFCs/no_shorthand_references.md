# No Shorthand References in Produced Artifacts

**Status:** planning
**Branch:** main
**Worktree:** none
**Mode:** interactive

## Original description

### No cryptic references across documents and resulting artifacts (code and docs, produced by following the skills in this project)

Principle to be implemented across the skills: Using short descriptive feature names is preferred, in code, docstrings, docs and design documents

We prefer very short names, beside IDs, for the artifacts (assumptions, principles, etc), created by `udesign` skill.
Within the `udesign` itself - it is acceptable to use artifact IDs, as specified previously.
But those ids got forgotten and cryptic later, therefore at later phases we prefer references by short names, where possible.

Major change must be in the `uplan` skill:
/up:plan must prefer to reference by these names, in the `uplan` skill, rather than IDs.
Nevertheless, we do not want hard enforcement of any behavior (e.g. using names or IDS),
The only hard requirement in uplan that we keep: for the plan referencing the design artifacts that are relevant for every plan point.

Make it clear in the `uplan` skill, so that is adds to every plan it creates:
Produced artifacts (doc, comments, code changes) should not use design ids or other abbreviated references from the design, that may appear cryptic later (to someone who never saw the RFC).

## Design

Stop design IDs *and* the new descriptive names from leaking into artifacts a stranger will read — code, comments, docstrings, project docs. IDs and names are both task-file-internal shorthand; produced artifacts must recite a design entity's full description. Make `udesign` attach a clarity-first name to every entity (so the task file can reference it concisely), have `uplan` carry that name in its prose and inject a produced-artifacts directive into every plan, and put the binding rule directly in the implementer agents — the agents that actually write code.

Files that change:

1. `plugins/up/agents/implementer.md` and `plugins/up/agents/implementer-sonnet.md` — add the produced-artifacts rule directly to the agents that write code. The rule lives where it is enforced.
2. `plugins/up/skills/udesign/SKILL.md` — output shape gains a clarity-first name field per entity; "max one sentence per definition" is replaced with a what / optional-why structure.
3. `plugins/up/skills/uplan/SKILL.md` — `Respects:` lines become `"name" (ID), "name" (ID)` (still task-file-internal); uplan injects a one-line directive into every plan it produces stating that produced artifacts must use the full description of any referenced design entity.
4. `plugins/up/skills/uexecute/SKILL.md` — implementer-dispatch template carries the full entity definitions (name + what + why), not ID-only lists, so the implementer has the description ready to recite.
5. `plugins/up/skills/uverify/SKILL.md` — minor: `CK<N> (IV1) — <prose>` form is preserved; the prose examples update to show natural use of names.

Entity definition shape:

```
- IV1 — <name> — <what: 1-2 sentences>. [Why: <max 2 sentences, IV/PC only, only when non-evident>]
- PC1 — <name> — <what: 1-2 sentences>. [Why: <…>]
- AS1 — <name> — <what: 1-2 sentences>.
- UK1 — <name> — <what: 1-2 sentences>.
```

Naming rule: clarity over brevity. Full English words; only widely-recognized abbreviations (`db`, `api`, `id`, `http`). Whatever length clarity requires.

Plan boilerplate directive (injected by `uplan` into every plan, also embedded in the implementer agents):

> Produced artifacts (code, comments, docs) must refer to design entities by their full description. Design IDs (IV*/PC*/AS*/UK*/PH*/RK*/CK*) and the udesign-attached short names are task-file-internal shorthand only.

Backwards compatibility: pre-existing RFC files (`Status: done`, old single-sentence ID format) are not retrofitted — same approach as `shorthand_ids`. New tasks created after this lands adopt the new format. In-flight tasks (Status ≠ done) pick up the new shape from their next stage onward.

Plugin version: patch bump in `plugins/up/.claude-plugin/plugin.json` per project rule.

TDD: no (doc-only plugin changes; no runtime code).

### Invariants

- IV1 — id-defined-in-udesign-only — Design IDs (IV*/PC*/AS*/UK*) are introduced exclusively inside `## Design` and its four subsections of the task file. Why: every other stage's output is at risk of reaching a stranger; the RFC's design section is the only place a reader has the definition in front of them.
- IV2 — name-is-mandatory-per-entity — Every IV/PC/AS/UK definition produced by `udesign` carries a clarity-first descriptive name in addition to its ID. Why: downstream stages can only reference by name if udesign provides one; an optional name silently becomes no name.
- IV3 — plan-respects-line-uses-name-plus-id — `uplan`'s `Respects:` lines and any in-prose reference to a design entity use the `"name" (ID)` form. Why: the ID-only form was the original cryptic-shorthand source we are removing.
- IV4 — produced-artifacts-recite-full-description — Code, comments, docstrings, and project docs (`README.md`, `CLAUDE.md`, `docs/**` outside the task's own RFC) produced by the implementer must not contain design IDs or the udesign-attached short names; references to a design entity recite its full description. Why: a stranger reading the repo months later has no RFC context to decode either `IV3` or `plan-respects-line-uses-name-plus-id`.
- IV5 — plan-design-traceability-preserved — Every plan bullet still maps to the design entities it preserves or relies on; only the citation form changes (name + ID instead of ID alone). Why: the only hard requirement the user explicitly asked to keep.

### Principles

- PC1 — clarity-over-brevity-for-names — Names spell out the concept in full English words; widely-known abbreviations only (`db`, `api`, …). Why: a name that needs a glossary is no better than the ID it replaced.
- PC2 — soft-touch-no-enforcement — Beyond IV1–IV5 the rules are guidance, not lint. Agents apply judgment when prose flows better without a name annotation. Why: the user explicitly rejected hard enforcement.
- PC3 — name-and-id-coexist-in-task-file — Inside the task file, IDs remain shorthand for cross-section references; names are layered on top, not a replacement. Why: the brevity gain `shorthand_ids` introduced inside RFCs is still valuable.

### Assumptions

- AS1 — no-retrofit — Pre-`done` RFC files are not retrofitted to the new shape; the change applies to tasks created or resumed after this lands.
- AS2 — agents-follow-skill-text — Planner / implementer / verifier agents read the updated SKILL.md text and apply it; no programmatic enforcement is added.

## Plan

Approach: One coordinated commit that embeds the produced-artifacts rule in the two implementer agent files, threads a name field through `udesign` output, and updates `uplan` / `uexecute` / `uverify` to consume and propagate it. All edits are doc-only; the changes are tightly coupled and split commits would obscure the dependency.

**Produced-artifacts directive (binding on this plan and every plan written hereafter):** Produced artifacts (code, comments, docs) must refer to design entities by their full description. Design IDs (IV*/PC*/AS*/UK*/PH*/RK*/CK*) and the udesign-attached short names are task-file-internal shorthand only.

### PH1 — embed the rule in the implementer agents and thread names through the upstream skills

- **1.1** `plugins/up/agents/implementer.md` (modify) and `plugins/up/agents/implementer-sonnet.md` (modify)
  - Both files, in their `## What you receive` block (around line 13 in each): update the bullet that today reads "Design IV (invariants), PC (principles), AS (assumptions)" to "Design Invariants, Principles, Assumptions — passed in full (name + description + why), not as ID-only lists."
  - Both files, in their `## Forbidden` section: add a new bullet — *"Using design IDs (IV\*/PC\*/AS\*/UK\*/PH\*/RK\*/CK\*) or the udesign-attached short names inside any produced artifact (code, comments, docstrings, project docs). IDs and short names are task-file-internal shorthand. When referencing a design entity in produced output, recite the full description from the dispatched entity bullet."*
  - Both files, in their `## Self-review checklist`: add one bullet — *"No design IDs or short names left in produced code, comments, or docs?"*
  - Respects: "produced-artifacts-recite-full-description" (IV4), "agents-follow-skill-text" (AS2)

- **1.2** `plugins/up/skills/udesign/SKILL.md:96-179` (modify)
  - Section `## ID conventions — define once, reference by ID` (lines 96-113):
    - Rule "Defined once with a full sentence at first appearance; every later mention is ID-only." (line 108) → reword: definitions carry a name and a what/why body (see new shape below); later mentions inside the task file may use ID alone, name alone, or both.
    - Rule "Max one sentence per definition." (line 109) → **delete**.
    - Insert new rule: "Every entity carries a clarity-first descriptive name in addition to its ID. Names use full English words; only widely-recognized abbreviations (`db`, `api`, `id`, `http`). Length is whatever clarity requires."
    - Keep existing rules about scoped numbering and the chat-expansion rule (lines 110-111).
  - Section `## Identifying invariants, principles, assumptions, unknowns` (lines 115-143):
    - Update each `<invariants>`, `<principles>`, `<assumptions>`, `<unknowns>` block's example bullets to the new shape (name + what + optional why). Example for IV:
      `- IV1 — dataset-must-not-import-from-training — The Dataset class must not import from training/. Why: training-side mutations would silently corrupt eval batches.`
  - Section `## Task-file output shape` (lines 157-179): update the markdown template to show `- IV1 — <name> — <what: 1-2 sentences>. [Why: <…, max 2 sentences, IV/PC only when non-evident>]`. Mirror for PC, AS, UK (AS/UK get no Why slot).
  - Section `## Rules` (lines 181-189): add one bullet — "Names are clarity-first: full words, popular abbreviations only, length follows clarity not brevity."
  - Respects: "id-defined-in-udesign-only" (IV1), "name-is-mandatory-per-entity" (IV2), "clarity-over-brevity-for-names" (PC1), "name-and-id-coexist-in-task-file" (PC3)

- **1.3** `plugins/up/skills/uplan/SKILL.md:64-119` (modify)
  - Section `## ID conventions in Plan` (lines 64-72):
    - Rule line 71 "References to Design entities use IDs (IV3, AS1, UK2) — never re-quote the full sentence." → reword to: 'References to Design entities use the `"name" (ID)` form (e.g. `"dataset must not import from training" (IV1)`). The name carries meaning to a future reader; the ID preserves traceability. Re-quoting the full sentence is still discouraged.'
  - Section `## Required contents` (lines 56-62), bullet "IV/PC/AS referenced by ID" → reword to "IV/PC/AS referenced by name + ID".
  - Section `## Format` (lines 84-119), in the Format block's `Respects:` example (line 94) replace `Respects: IV2, AS1` with `Respects: "single-tx writes" (IV2), "upstream-email-utf8" (AS1)`. Mirror in any other example lines that show ID-only citations.
  - Insert a new top-level section `## Produced-artifacts directive` between `## Required contents` and `## ID conventions in Plan`. Body: every plan `uplan` writes must include, near its `Approach:` line, the verbatim directive: *"Produced artifacts (code, comments, docs) must refer to design entities by their full description. Design IDs (IV\*/PC\*/AS\*/UK\*/PH\*/RK\*/CK\*) and short names are task-file-internal shorthands only. Reason: a stranger reading the codebase later has no RFC context to decode either form."
  - Update Format example to show the directive line under `Approach:`.
  - Self-review checklist item 5 (line 136) "each IV and AS has a referencing bullet somewhere (by ID)" → "(by name + ID)".
  - Respects: "plan-respects-line-uses-name-plus-id" (IV3), "plan-design-traceability-preserved" (IV5), "soft-touch-no-enforcement" (PC2)

- **1.4** `plugins/up/skills/uexecute/SKILL.md:88-119` (modify)
  - Section "Pass in the dispatch prompt" (lines 88-104): replace bullet "`### Invariants` (IV), `### Principles` (PC), `### Assumptions` (AS) from `## Design`" (line 91) with "`### Invariants`, `### Principles`, `### Assumptions` from `## Design`, passed in full (name + what + optional why) — not as ID-only lists."
  - Section "Dispatch prompt skeleton" (lines 108-119): replace the three lines
    ```
    Invariants: <IV1, IV2, ...>
    Principles: <PC1, PC2, ...>
    Assumptions: <AS1, AS2, ...>
    ```
    with a single block that embeds the full Design subsections verbatim, e.g.:
    ```
    Invariants:
      - <verbatim ### Invariants bullets, name + what + why>
    Principles:
      - <verbatim ### Principles bullets>
    Assumptions:
      - <verbatim ### Assumptions bullets>
    ```
  - Add one line under the skeleton: "Implementer recites the full description of any referenced entity in produced code, comments, or docs; the implementer agent's `## Forbidden` section binds this rule."
  - Respects: "produced-artifacts-recite-full-description" (IV4), "agents-follow-skill-text" (AS2)

- **1.5** `plugins/up/skills/uverify/SKILL.md:18-46` (modify)
  - Section "Phase 1 — Build the checklist" (lines 16-52): no structural change; the `CK<N> (IV1) — <prose>` form is preserved. Update the prose in example CK lines (lines 40-42) to use the descriptive name in the prose body, e.g.:
    `- CK5 (IV1) — "dataset must not import from training" — grep "from training" src/dataset/ → empty`
  - Add one short rule sentence near line 24: "CK lines retain the `(ID)` annotation; the prose may state the entity name for clarity but no extra annotation is required."
  - Respects: "plan-design-traceability-preserved" (IV5), "soft-touch-no-enforcement" (PC2)

- **1.6** `plugins/up/.claude-plugin/plugin.json:3` (modify)
  - Bump `"version"` from `0.3.8` to `0.3.9` (patch, per project rule).
  - Respects: (none — administrative)

- Commit: `feat(plugins/up): names alongside IDs in udesign/uplan/uexecute/uverify; produced-artifacts rule in implementer agents; bump 0.3.9`

### Risks / rollback

- RK1 — Existing in-flight RFC files (e.g. `parallel_phase_exec.md`, `interface_first_parallel.md` if any are non-`done`) reference entities ID-only; new uplan instances picking them up will not retroactively gain names. Mitigation per "no-retrofit" (AS1): the new shape applies from the next stage forward; existing entries are not rewritten. Rollback: revert the single commit; no data migration involved.


## Verify
<empty — filled by up:uverify>

## Conclusion
<empty — filled by up:ureview>

### Hands-off decisions
<empty — populated only when Mode is hands-off>

### Deferred (needs user input)
<empty — populated only when Mode is hands-off and a choice had no conservative default>
