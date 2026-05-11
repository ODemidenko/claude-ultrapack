## No cryptic references across documents and resulting artifacts (code and docs, produced by following the skills in this project)

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

