# ultrapack

Claude Code plugin for spec-driven, git-centered development. Distributed as a GitHub-native marketplace: the repo root is a marketplace (`.claude-plugin/marketplace.json`) that contains one plugin, `up`, under `plugins/up/`.

## Repo layout

- `.claude-plugin/marketplace.json` — marketplace manifest (lists `up`)
- `plugins/up/.claude-plugin/plugin.json` — plugin manifest
- `plugins/up/{skills,commands,agents,hooks}/` — plugin contents
- `docs/RFCs/*.md` — task files (design + plan + conclusion per task)
- `README.md`, `CLAUDE.md` — repo docs

Everything under `plugins/up/` loads into Claude Code. Everything outside (`docs/`, README, CLAUDE.md) is repo-only.

Internal plugin structure and skills+agents inter-dependencies is described in the file: docs/plugin_schema.md.
Upon major interdependencies updates (resulting in how skills interact), after `uexecute` finishes: propose updating this file.

## Naming

Internal plugin name: `up`. Slash/skill invocations use the `up:` prefix: `/up:make`, `up:udesign`, `up:reviewer`. Process skills are `u`-prefixed (`udesign`, `ubrainstorm`, `uplan`, `uplan-freeform`, `uexecute`, `uverify`, `ureview`, `udebug`, `udocument`) to dodge collisions with Claude Code built-ins.

`ubrainstorm` is a standalone challenger to `udesign` for hard, open-ended design tasks where the user doesn't yet know the right shape of the answer — invoked directly by the user, never from the `/up:make` chain. It deliberately drops the formal IV/PC/AS/UK spec format to keep the model's attention on the design itself rather than on format compliance.

`uplan-freeform` is the parallel challenger to `uplan` — invoked directly, never from the `/up:make` chain. It deliberately drops the PH/RK/IF interface graph and IV/PC/AS/UK spec-coverage scaffolding in favor of a plain-markdown plan one altitude above code (files, symbols, behaviors, phase order — no line numbers, no function bodies). Use it when the formal scaffolding would weigh more than the task warrants, or downstream of a `ubrainstorm` brief that needs only a written plan rather than the full `/up:make` chain.

## Design principles

- **Minimal** — only skills we actually use; no speculative additions
- **Doc-only** — no runtime code, no unit tests; verification is install-and-invoke

## Specific requirements when improving this project internals:
- **we require for the skills descriptions to be as crisp as possible.** When making any change to the skills - validate if it is not duplicating instructions in the neighbor skills. Skills must follow DRY principle, and required behavior should be usually define only in a single place, where it suits most. Plan for this best placement rigorously, and consult with the user.
*Duplicating instructions across multiple associated skills/agents - is the biggest source of bloat and potential inconsistencies* (as later, such a duplicated instruction gets updated only in a single place).
- when applying uplan skill for this project changes: propose the actual wording to be used. Ignore the usual plan instructions for being a "minimum information an implementer needs" (in this markdown-only project it is irrelevant).

## Versioning

Plugin version lives in `plugins/up/.claude-plugin/plugin.json`. Always bump the patch digit (`x.y.Z`) when merging, finalizing, or otherwise landing changes on `main`. Default to patch; ask before bumping minor (`x.Y.z`) or major (`X.y.z`).

## Local plugin update

This repo is consumed as a live local-marketplace install for Claude code. 
After **any** edit to a file under `plugins/up/`:

- surface the following reminder to the user verbatim, in the same response that performed the edit:

> REMINDER: each time you edit a file under plugins/up/ in your working repo, you'll need to rerun /reload-plugin (and the marketplace must allow auto-update)

Edits outside `plugins/up/` (e.g. `docs/`, `README.md`, this file) do not require the steps above
