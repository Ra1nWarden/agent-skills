---
name: archive-blueprint
description: Use when the user asks to archive one or more blueprints, explicitly declares a project or workstream complete, or the finishing implementation commits are successfully pushed to the project remote. Moves every relevant research, spec, plan, and review artifact out of active discovery.
argument-hint: <filename, slug, workstream, or completed project>
allowed-tools: Read Glob Grep Bash(ls *) Bash(mkdir *) Bash(mv *) Bash(find *) Bash(git add*) Bash(git commit*) Bash(git push*) Bash(git pull*) Bash(git remote*) Bash(git status*) Bash(echo *) Bash(basename *) Bash(pwd) Bash(date *) Bash(cd *)
---

# Archive Blueprint

Move completed blueprint artifacts from the active `research/`, `spec/`,
`plan/`, and `review/` folders into the project's `archive/` folder. Archived
artifacts are retired from future blueprint discovery.

Support both completion-aware batch archival and direct requests to archive a
named artifact.

## Completion Signals

Treat either event as authorization to archive the relevant blueprint set:

1. The user explicitly says that the whole project or a named workstream is
   done.
2. The agent successfully pushes the finishing implementation commit or commits
   for the approved work to the **project repository's** remote.

Use a push as a completion signal only when the surrounding workflow establishes
that it finishes the approved work. Do not infer completion from an arbitrary
or intermediate implementation push. Blueprint-repository pushes,
planning-only pushes, and failed project-repository pushes are never completion
signals.

When a signal is clear, proceed without asking for a second confirmation. If it
is unclear whether a push finishes the work, do not archive on the basis of the
push alone.

## Environment Setup

Follow the `blueprints` skill: verify `$BLUEPRINTS_DIR`, derive `$PROJECT`, and
use its commit-on-write protocol.

## Resolve the Archive Set

Search only Markdown files in the current project's active `research/`,
`spec/`, `plan/`, and `review/` directories. Never select a file already in
`archive/`.

Choose the scope from the completion signal or direct request:

- **Whole project completed:** If the user unambiguously declares the whole
  project done, select every active blueprint for `$PROJECT`.
- **Workstream completed:** Start with the implemented plan or spec. Use its
  slug, title, stated scope, document links, and the current implementation
  context to identify the complete artifact chain. Include the implemented
  plan/spec, research it depends on, and wave or conformance reviews produced
  for that work. Include other artifacts only when their content establishes
  that they belong to the same workstream. Leave unrelated active efforts in
  place.
- **Named archive request:** Search all active folders for the requested exact
  filename or slug. A request may intentionally match several related files;
  read the plausible matches and select the set that belongs to the requested
  topic.
- **Archive request without a target or completion signal:** List active
  blueprints grouped by type, ask the user which artifact or workstream to
  archive, and stop.

Read every plausible match in full before deciding. If a material ambiguity
remains about whether an artifact belongs to the completed work, present the
proposed set and ambiguous candidates, ask the user to confirm, and stop. Do not
archive unrelated files merely because they share the project directory.

If the resolved set is empty, report that there are no active relevant
blueprints and make no commit.

## Archive the Set

Treat the resolved set as one batch:

1. Before moving anything, verify that every source is a distinct Markdown file
   under an active folder and that no destination filename already exists in
   `$BLUEPRINTS_DIR/$PROJECT/archive/`. Stop rather than overwrite or rename an
   existing archive artifact.
2. Check the blueprint repository status. Do not include or disturb unrelated
   user changes; stop if the shared commit-on-write command would capture them.
3. Create `$BLUEPRINTS_DIR/$PROJECT/archive/` if needed, then move every
   resolved file into it. Perform all preflight checks before the first move.
4. Create one blueprint-repository commit for the batch using
   `archive($PROJECT): complete <workstream>` or
   `archive($PROJECT): complete project`, then push it with the `blueprints`
   commit-on-write protocol.
5. Build a remote archive link for every moved file using the pushed branch and
   the `blueprints` remote-link rules.

If a move unexpectedly fails after the batch starts, do not commit a partial
archive. Report the exact moved and unmoved paths so recovery does not risk
overwriting user work.

## Confirm

Tell the user:

- Which completion signal or direct request triggered archival.
- Every original path and archive path.
- A clickable remote link for each archived file when available.
- That the files no longer appear in active blueprint listings.
