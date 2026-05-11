Between design and plan there can be a longer break. Propose design commit upon udesign finish, to the current branch, no branch creation at this point.

Implementation plan must rather tell what functions/classes to fix (stable references), instead of referencing the current line numbers (most likely to get obsolete).
Propose to commit the RFC document with the plan in it

the /up:make must propose to create git worktree and branch when the execution starts, at /up:uexecute, no earlier, as otherwise code may substantially evolve between the planning and executing phase.