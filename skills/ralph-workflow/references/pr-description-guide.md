## PR Description Guidelines

PR descriptions must include the sections below. They apply even for very short PRs. Include all sections even when creating multiple linked PRs (each PR description should have its repo's point of view.)


**Overview**:
- Explain the project succinctly, adding enough context for people who don't know about the project
- Mention any underlying motivations and potential impacts/benefits
- If security risk is Medium or High (according to `Security assessment` section), mention that
  security review is advised and point to the `Security assessment` section for further details
- At the end, add the best possible screenshot that represents the work done (e.g. screenshot of
  the UI for UI work, benchmark chart for performance work, data chart for data work)

**Linked PRs**:
- List all linked PR URLs, if any (e.g. working across multiple repos)
- If there are no linked PRs, skip the section's title and add small note saying so.

**How to review (for human reviewers)**:
- Categorize all files changed into categories (e.g. documentation, core logic, utils, tests, CI)
- For each category, reread the files to come up with the best file review order
- Add a table with these columns:
  * _Category_: The file category
  * _Changes_: Number of files and lines changed (use format: `N files, +X, -Y`, where X
    are lines inserted and Y lines deleted)
  * _Files (in suggested review order)_: List files for reviewing with review order represented
    by arrows. Use relative paths. Always mention the key files by name, but glob patterns are
    allowed, especially if there are too many paths to list (use format:
    `p1/f1.ext (+X -Y) → p1/f2.ext (+X -Y) → p2/*.ext (+X -Y)`, where X/Y are lines
    inserted/deleted for each specific file or glob pattern).
- Below the table, say that they should first read the full PR description, mention the
  `How to test (for human reviewers)` section at the end of the PR description, and
  suggest a file category review order (typically documentation → core logic → ...)

**How it was developed**:
- Say that this was developed using the Ralph Wiggum Loop workflow
- Point to the workdocs and explain each one succinctly, but avoid repeating their contents
- Mention what agent harness was used (e.g. Claude Code CLI)
- Mention what LLM model was used for each workflow phase

**How it was reviewed and tested**:
- Provide enough information about how this was reviewed and tested (e.g. self-reviews, UAT)
- Mention what tools were used to self-verify (e.g. Chrome DevTools MCP)
- Add test coverage table per file category, including columns:
  - _Category_: Same categories defined in `How to review (for human reviewers)` section
  - _Statements_: `X/Y (Z%)`, X being the covered statements, Y total statements, and Z% the
    statement coverage percentage
  - _Branches_: `X/Y (Z%)`, X being the covered branches, Y total branches, and Z% the
    branch coverage percentage
  - _Notes_: Summarize testing approach used, what is covered, and any known gaps

**Changes**:
- Add a flat list of bullets summarizing what was done (and not how it was done). Don't mention
  file names – just describe what was done. Don't nest bullets. Make each bullet short (one line).
  Be exhaustive.
- Add any other interesting screenshots at the end of this section

**Security assessment**:
- Summarize all security aspects considered. Mention any pre-existing concerns and whether they
  were handled. Mention any new dependencies and systems and how they were
  secured or deemed to be safe. Mention any risk mitigations added
- Classify this PR in the risk scale below, and explain reasoning for the choice:
  1. Low (should be fine without security review)
  2. Medium (security review advised)
  3. High (security review advised, reconsider if it really should be shipped)

**How to test (for human reviewers)**:
- Add flat list with granular steps and checkboxes, including any setup steps and everything they
  should try out
