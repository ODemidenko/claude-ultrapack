## Free-form input

There may be a case when user provides a file as input to the whole "ultrapack" system,
this is usually a vague feature draft.

update the `make.md` command description and, if required all the skills it references, so that they can accommodate scenario when `make` command or udesign skill are called with a file reference.

Scenarios to be supported:
- `make.md` is called with a file name that is out of "RFCs" folder. *Example of such a file is this one.*
    - Then, reuse the same filename for a new file, created in the `RFCs` folder. This name must be the "slug" 
    - And, to distinguish filenames, the original filename must be prefixed with `wip_`
- `make.md` command is called with a file that is already in `RFCs` folder. In this case, `make` command edits this file in place.

In both cases:
file content is regarded as a first draft, and its original wording, must be put, with minimal reasonable changes (grammar fixes / converting all headers to start from H2), into the "## Original description" in the RFC description that `make.md` command creates. 

## Design step
  - `design/SKILL.md` must expect that input file *may, but is not required* to contain the "## Original description" heading
  - design skill may and is expected to change the original description, in case any clarifications come up during the design process
  Though, in case there were no major change in direction, it should rather keep the original structure/wording, as the user would prefer to recognize the initial information exactly in his own words. And there is a further "## Design" section with a goal to disambiguate.
  - under the "## design" part that the design skill is filling, there can be a reasonable duplication/restructuring of the information, provided in the "

## Future steps
Check if any other files have rigid reference to "## Design" part in the RFC doc, which may lead to them omitting the original description.
Wherever the "## Design" part is visible, the "## Original description" part must be visible as well.