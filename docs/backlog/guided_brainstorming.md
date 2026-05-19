# New ubrainstorm skill

see @plugins/up/skills/udesign/SKILL.md.
This skill currently conflates the precise format and orchestration requirements (that are likely to be project specific), with ideas about a robust design process.
As a result, it gives designs that have awesome structure (enterprise-level uniformity), 
but oftentimes design quality is higher, in case of a short free-form requests to the same (SOTA) LLM models.

I want you to extract purely the ideas and behavior valuable for design/brainstorming of the content, to avoid overloading Agent with excessive format-level instructions.
This must be put into a new skill ubrainstorm, that will compete with udesign for becoming a skill of choice for working on the hardest designs, where the user doesn't know the best approach yet.

**Give exactly the right level of instructions**
Your are required to give guidelines that ensure a better quality design process by the SOTA LLM model (yourself, or future version of yourself).
And as this model is probable to concentrate on following the every skill detail (at the expense of attention, devoted to the complex system being designed) - we want this skill to be high-level,
and avoid imposing hard constraints on minor details: to avoid driving model attention towards following the design process, instead of concentrating on the outputs.

Keep in mind this Anthropic quote on how good instructions are formed (system prompts and skills require the same trade-off):
"""
System prompts should be extremely clear and use simple, direct language that presents ideas at the right altitude for the agent. The right altitude is the Goldilocks zone between two common failure modes. At one extreme, we see engineers hardcoding complex, brittle logic in their prompts to elicit exact agentic behavior. This approach creates fragility and increases maintenance complexity over time. At the other extreme, engineers sometimes provide vague, high-level guidance that fails to give the LLM concrete signals for desired outputs or falsely assumes shared context. **The optimal altitude strikes a balance: specific enough to guide behavior effectively, yet flexible enough to provide the model with strong heuristics to guide behavior.**
"""

**Do not invest effort into orchestrating with other skills**
No cross-links with the other skills is required. Brainstorming may be used on its own (expect it to receive either chat inputs or short briefs, to be extended into full designs).
Though, you may re-use other skills (e.g. _brevity.md may be useful, or simply borrow its ideas and vendorize, by incorporating their wording directly into this skill).

**Persona/attitude:**
Also, model must be more inclined into performing the guided brainstorming with the behavior described below 
(think Dr House persona - that is kind of ruthless professional honesty and agency that we want to get from the brainstorming, guided by this skill).

## Guided brainstorming / grilling user

**Proposals below may be excessive, the core motivation is chaing udesign skill to be more assertive with best practices + YAGNI and push user to invest more effort into building a future proof design**


Allow Agent create catchy (easily memorizable) names to use for the new functionality and its parts (e.g. `reason codes`).
Agent should validate with the user that those terms are clear, before heavily reusing them (do not emphasize this validation, apply you judgement to guess what glossary requires validation)


### Agent at design stage must have grilling behavior, being ruthlessly honest, and highly interrogative.
This is the expected context for the udesign skill:

User provided some task description. 
He almost certainly overlooked smth in the task as he posed.

Agent role is to understand what he is missing (task may get much larger), and through YAGNI to reduce the task to the necessary minimum
(task may get much smaller).

This is recommended verbatim addon to udesign skill!

**Grilling protocol**
Interview use relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies one by one.
For each question, you must provide your recommended answer based on the current codebase and best practices.
- Do not move to the next topic until the current one is definitively resolved.
- If I provide a vague answer, push back for more detail.
Once a shared understanding is achieved, summarize the design concept before we proceed to the next phase.
