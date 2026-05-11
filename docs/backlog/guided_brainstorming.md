## Guided brainstorming / grilling user

**this is probably excessive, just update udesign skill to be more assertive and push back for detail**
**allow udesign any structure within the "design" part"**
**invariants are misleading, those must be optional and probably better defined**

allow him using "tasks" / functionality parts, and create catchy names to use (e.g. `reason codes`)



# original wording
-  `/up:make` get the task description and puts it into "original description"
-  From it, it must be clear if user wants guided brainstorming/ grilling. Keywords that trigger this behavior: brainstorm / grill / review/ criticize.

### Grilling behavior
User provided some task description. 
He almost certainly overlooked smth in the very task. 
Agent role is to understand what he is missing (task may get much larger), and through YAGNI to reduce the task to the necessary minimum
(task may get much smaller).

This is actually a mode of behavior for udesign skill, and must be added as an aggressive "add-on" there.

### Grilling protocol
Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies one by one.
For each question, you must provide your recommended answer based on the current codebase and best practices.
- Do not move to the next topic until the current one is definitively resolved.
**- If I provide a vague answer, push back for more detail.**
Once we have a shared understanding, summarize the design concept before we proceed to the next phase
