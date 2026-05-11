Context:
Between design and plan there can be a longer break. 
Desired behavior:
Propose user to commit design upon udesign finish, to the current branch, no branch creation at this point.

Context:
between planning and implementation there may be another (strictly orthogonal) changes introduced in the code.
Desired behavior:
Implementation plan must rather tell what functions/classes to fix (stable references), instead of referencing the current line numbers (most likely to get obsolete).
Propose the user to commit the RFC document with the plan in it, before execution.

Verify:
the /up:make must propose to create git worktree and branch only when the execution starts, at /up:uexecute, no earlier, as otherwise code may substantially evolve between the planning and executing phase.

Executor:
Executor must prefer NOT commiting the change, in case of single-phase changes. The goal is to allow the user to validate the code before it was committed.
I.e. trigger implementer with `commit: defer` mode, in case of a single phase plan.
And propose user to review the code, upon completion. 