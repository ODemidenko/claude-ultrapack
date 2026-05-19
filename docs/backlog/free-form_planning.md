# New uplan-easy skill

Current uplan skill probably conflates the precise format and orchestration requirements (that are likely to be project specific), with ideas about a robust planning of the required changes in the code.
As a result, it gives plans that have awesome structure (enterprise-level uniformity), 
but plan quality is probably better, in case of a short free-form requests to the same (SOTA) LLM models.

I want you to extract purely the ideas and behavior valuable for planning code changes, to avoid overloading Agent with excessive format-level instructions.
This must be put into a new skill `uplan-easy`, that will compete with `uplan` for becoming a skill of choice for better planning the required changes

**I want you to put in this skill exactly the right level of instructions**
Your are required to give guidelines that ensures a better output by the SOTA LLM model (yourself, or future version of yourself).
And as this model is probable to concentrate on following the every skill detail (at the expense of attention, devoted to the task at hand) - we want this skill to be reasonably high-level,
and avoid imposing hard constraints on minor details: to avoid driving model attention towards following the process, instead of concentrating on the results.

Keep in mind this Anthropic quote on how good instructions are formed (system prompts and skills require the same trade-off):
"""
System prompts should be extremely clear and use simple, direct language that presents ideas at the right altitude for the agent. The right altitude is the Goldilocks zone between two common failure modes. At one extreme, we see engineers hardcoding complex, brittle logic in their prompts to elicit exact agentic behavior. This approach creates fragility and increases maintenance complexity over time. At the other extreme, engineers sometimes provide vague, high-level guidance that fails to give the LLM concrete signals for desired outputs or falsely assumes shared context. **The optimal altitude strikes a balance: specific enough to guide behavior effectively, yet flexible enough to provide the model with strong heuristics to guide behavior.**
"""

**Do not invest effort into orchestrating with other skills**
No cross-links with the other skills is required. This skill may be used on its own, though it will be usually applied to the outputs of the ubrainstorm skill.
Nevertheless you may re-use other skills (e.g. _brevity.md may be useful, or simply borrow its ideas and vendorize, by incorporating their wording directly into this skill).

## Required outputs
My expectation is that final plan will be always written down to disk.
If input was in a file - propose storing into the same file, or make your proposal based on the project conventions.

## How to deal with this skill creation
Try to follow the gist of ubrainstorm skill, and guide me with the most valuable high-level skill description for the planner agent.
I also want this skill to be designed using the same principles, as ubrainstorm uses.
Also, read the prompt of the built-in Claude Code "plan" agent, what you need to produce - is a competing implementation, that will be a bit more high-level in its outputs (stop on the level of a function/module or exact behavior , rather than detailing the code).