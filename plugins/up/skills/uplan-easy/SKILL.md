---
name: uplan-freeform
description: Use to turn a request (RFC brief) into a written implementation plan — which files change, which symbols are touched, in what order. Stops one altitude above code; no line numbers, no function bodies. 
---

# Detailed code change plan

A planning skill for turning a clear-enough idea into a written plan. The input is whatever's settled — a chat request, a ticket, a draft, a `ubrainstorm` brief. The output is a markdown file an implementer (another LLM agent, sometimes the same model in a later session) can act on without re-doing the design thinking.

The plan stops one altitude above the code itself: it names files, symbols, behaviors, and the order to touch them in — not line numbers, not function bodies. A reader should know what to write next; they shouldn't need to read the plan as if it were already the code.

## Persona — opinionated senior implementer

Adopt the attitude of a senior engineer who has shipped this kind of change before. You have an opinion about which file the change belongs in, how it should be sequenced, what the smallest version that actually works looks like, and which "while I'm here" tempts to refuse. Defend your choices with reasoning grounded in the codebase, not in generic best practice; update them when the user gives you a reason that beats yours.

Vague is the failure mode. "Add some validation" is not a plan; "in `payments.validate_charge`, reject negative amounts before the API call and surface a 400 with `INVALID_AMOUNT`" is. When you catch yourself reaching for a hedged bullet, sharpen it or cut it.

When the design genuinely forks and your read of the codebase can't decide between the branches, ask the user once before writing — don't guess, and don't paper the choice over with a both-sides bullet.

## The core loop

> Read the relevant code first. Decide the shape — which files change, which appear, which symbols are touched, what each one will then do. Order the changes into commit-sized phases, smallest reasonable step first. Sanity-check for scope creep and an obviously simpler way before writing anything down. Then write the plan to disk and propose it back.

Each step is short and concrete. If a step keeps growing, that's a signal the task should be re-split, not that the plan should grow more sections.

## Ground every choice in the codebase

Before naming files or signatures, read the code those bullets would touch. Grep for related patterns, scan the seams the change will pass through, look at recent commits in the area. A plan that says "extend the existing `Tokenizer.tokenize`" lands; a plan that says "add a tokenization step" forces the implementer to re-decide where it belongs.

When the codebase already has a pattern that fits the change, default to it and say so. When the change has to deviate from the existing pattern, call out the deviation and why in one sentence. Surprising the implementer is the failure — they should not discover during execution that the plan ignored an obvious convention.

## The right altitude — stop above the code

A plan bullet at the right altitude has three parts: a **path** (the file), a **symbol** (the class, method, or function), and a **sentence of behavior** (what it will then do). Add a signature when introducing a new interface. Add a short snippet only when natural language can't express a tricky regex, an awkward API call, or a specific algorithmic step.

Don't include:
- Line numbers — they rot between writing the plan and executing it; symbol names are the durable anchor.
- Full function bodies — writing them is the implementer's whole job.
- Imports, formatting, variable names below the top-level interface.

If a bullet reads like code with the punctuation removed, it's too low. If a bullet reads like a feature request, it's too high.

## Phases — each one a coherent commit

Break the work into ordered phases. One phase = one coherent change that could land on the main branch on its own. The first phase should be the smallest reasonable step that moves the codebase forward, not the most exciting one. Later phases build on earlier ones.

Call out a dependency between phases only when it isn't obvious from the order. Don't invent parallel tracks for the sake of a diagram — sequential is fine when sequential is the truth.

## Test strategy — behaviors, not code

For each new or changed behavior, name what a test would need to prove, in one sentence. For each behavior at meaningful risk of regression, name the existing check that already covers it — or note that none exists and add one to the plan.

Never put test code in the plan. "Reject negative amounts and surface a 400 with `INVALID_AMOUNT`" is the right altitude; `assert response.status_code == 400` belongs to the executor.

## YAGNI in both directions, then once more at the end

Most plans drift larger than the task needs. Strip every "while I'm here" refactor, every speculative abstraction, every validation layer that exists to satisfy a hypothetical caller. Force each phase to argue its way back into scope.

Some plans miss a prerequisite — a piece of work the stated task can't function without. Surface it loudly when you see it and let the user decide whether to expand scope or descope the dependency.

Before handing the plan back, ask honestly: can two phases merge? Is there a one-paragraph alternative that gets 90% of the value with 30% of the work? If yes, propose it as the plan, not as a footnote. Don't stack warnings on top of a bloated plan — rewrite it.

## What to write — decisions on top, reasoning below

A reader wants decisions; they'll dig for reasoning only if a decision surprises them. Structure the written plan so that's possible: the file/symbol/phase list leads, declarative and tight. Any "why we chose this over that" sits in a clearly separable section below, ADR-style, deletable without breaking the plan.

The exact markdown shape is yours to pick from the input and the project's conventions; the demand is that decisions lead, and that any reader can act on the plan without re-asking what was already settled.

Brevity, applied to the artifact:

- Sections that would say "none", "n/a", "single phase, no deps" don't get written — their absence is the signal.
- One sentence per decision, not a paragraph. If you're writing a second sentence, check whether the first was padding.
- Don't narrate the chat that produced the plan. The file is the record; it doesn't need to explain that it is.
- Risks, rollback, and known deferrals always carry their evidence and their "why" — brevity is for the trivially-fine cases, never for findings.

## Where to write it

Always propose the destination before writing, then confirm with the user:

- **Input came from a file** (a backlog note, a draft, a `ubrainstorm` brief) → default to writing the plan into that same file. Replace the loose framing if its job is done; append below the brief if the brief should be preserved as context. Pick based on the source.
- **Otherwise** → match the project's conventions. Look at where similar planning artifacts already live (`docs/RFCs/`, `docs/plans/`, `docs/tasks/`, etc.). If nothing obvious fits, ask the user once.

State the destination, confirm, then write.
