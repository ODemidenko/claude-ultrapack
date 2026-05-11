# ultrapack

Claude Code plugin for spec-driven, git-centered development. Distributed as a GitHub-native marketplace: the repo root is a marketplace (`.claude-plugin/marketplace.json`) that contains one plugin, `up`, under `plugins/up/`.

## Repo layout

- `.claude-plugin/marketplace.json` — marketplace manifest (lists `up`)
- `plugins/up/.claude-plugin/plugin.json` — plugin manifest
- `plugins/up/{skills,commands,agents,hooks}/` — plugin contents
- `docs/RFCs/*.md` — task files (design + plan + conclusion per task)
- `README.md`, `CLAUDE.md` — repo docs

Everything under `plugins/up/` loads into Claude Code. Everything outside (`docs/`, README, CLAUDE.md) is repo-only.

## Naming

Internal plugin name: `up`. Slash/skill invocations use the `up:` prefix: `/up:make`, `up:udesign`, `up:reviewer`. Process skills are `u`-prefixed (`udesign`, `uplan`, `uexecute`, `uverify`, `ureview`, `udebug`, `udocument`) to dodge collisions with Claude Code built-ins.

## Design principles

- **Minimal** — only skills we actually use; no speculative additions
- **Doc-only** — no runtime code, no unit tests; verification is install-and-invoke

**Note: we require for the skills descriptions to be as crisp as possible.**

## Versioning

Plugin version lives in `plugins/up/.claude-plugin/plugin.json`. Always bump the patch digit (`x.y.Z`) when merging, finalizing, or otherwise landing changes on `main`. Default to patch; ask before bumping minor (`x.Y.z`) or major (`X.y.z`).

## Local plugin update

This repo is consumed as a live local-marketplace install for Claude code. 
After **any** edit to a file under `plugins/up/`:

- surface the following reminder to the user verbatim, in the same response that performed the edit:

> REMINDER: each time you edit a file under plugins/up/ in your working repo, you'll need to rerun /reload-plugin (and the marketplace must allow auto-update)

Edits outside `plugins/up/` (e.g. `docs/`, `README.md`, this file) do not require the steps above
