## Vertical Stages planner

The plan skill (uplan/SKILL.MD) must consider the following:

## Work in reasonably big phases
Generally, Ai coding agents work better, when they have broader context, i.e. work is not split into too small chunks.
Therefore, we want to dispatch implementing agent per phaze, where one phase must be reasonable big (google with research agent: what amount of file edits may fit into 100k tokens).
It is ok to have single-phase plan, containing 10 files, or even dozens files, in case of a trivial edit expected.

## Work in vertical slices
Also, we prefer every phase to be end to end, across all the application layers (parts), to allow some user-testable result, at least partial, instead of implementing the functionality layer by layer.

For this sake, project can have intermittent phases that are partially rewritten by the next phase, in case this allow reasonable user test of a partial result.



# using uexecute dispatching with user awareness
Ask user if he wants to check results in phases. 
If yes: executoer must trigger 1 phase and trigger verifier (and next it tiggers reviewer) for this stage. 
The document must track the stage (what "phase" is being implemented/reviewed). For this sake, implemented phase must be marked as ("code completed", "verification completed", "completed") after implementer/verifier/reviewer.
Then, user is informed that the phase is completed.

Upon user review - he may ask for another phase to be continued.
