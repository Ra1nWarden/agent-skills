---
name: blueprints
description: Use when working with persistent planning artifacts in $BLUEPRINTS_DIR, including research briefs, design specs, implementation plans, code reviews, and archived blueprints. Defines the layout, naming, and commit conventions other blueprint skills rely on.
---

# Blueprints

Blueprints are persistent, git-tracked design artifacts (research briefs,
specs, plans, reviews) stored outside project repos in a dedicated
blueprints repository.

## Environment Check

Before any blueprints operation, verify `$BLUEPRINTS_DIR` is set:

```sh
echo "$BLUEPRINTS_DIR"
```

If empty, STOP and tell the user:
> `BLUEPRINTS_DIR` is not set. Run `./install` from your dotfiles repo or add
> `export BLUEPRINTS_DIR=~/path/to/blueprints` to `~/.zshrc.local`, then
> restart your shell.

## Project Derivation

```sh
PROJECT=$(basename "$(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null \
  | sed 's|/\.git$||; s|/\.bare$||')" 2>/dev/null || basename "$(pwd)")
PROJECT=${PROJECT#.}
```

The last line strips a leading dot so hidden-directory repos map to a normal
blueprints folder (e.g. `.dotfiles` → `dotfiles`).

## Human-Readable Summary

Every active blueprint must begin with a concise `## Summary` immediately after
its top-level title, before metadata or detailed sections. This is the first
section a human reader should see.

- Use plain language, normally 3–7 bullets and no more than about 150 words.
- State the bottom line rather than repeating the document's detailed evidence.
- For research, summarize what was investigated, the most important findings,
  and why they matter.
- For designs, summarize the problem, recommendation, key trade-offs or risks,
  and any decision needed.
- For plans, summarize the intended outcome, approach, major phases, and
  important constraints or approval state.
- For reviews, summarize the verdict, highest-priority findings, and required
  next action.

Refresh the summary after every material update so it remains consistent with
the body. When updating an active blueprint that has no summary, add one as part
of that update. Do not rewrite untouched or archived blueprints solely to add a
summary.

## Reuse Existing Blueprints

Before creating any blueprint, check the current project's active `research/`,
`spec/`, `plan/`, and `review/` directories for an existing artifact with the
same topic and purpose. Use filenames plus document titles, scopes, goals, and
other relevant content to identify plausible matches.

Read the full content of every plausible match before proceeding. If an active
blueprint already covers the requested artifact, update the most directly
relevant file in place. Preserve its path and original timestamp; do not create
a second blueprint. Create a new timestamped file only when no active blueprint
matches.

Do not include `archive/` in this check. Archived blueprints have been
explicitly retired from future workflow context. This reuse-before-create rule
applies to every blueprint-producing skill, even when a downstream workflow
also instructs the agent to generate a filename.

## Directory Layout

```
$BLUEPRINTS_DIR/<project>/research/   # codebase research briefs
$BLUEPRINTS_DIR/<project>/spec/       # design specs
$BLUEPRINTS_DIR/<project>/plan/       # implementation plans
$BLUEPRINTS_DIR/<project>/review/     # code review output and conformance reports
$BLUEPRINTS_DIR/<project>/archive/    # consumed blueprints (all types)
```

Create on first write: `mkdir -p "$BLUEPRINTS_DIR/<project>/<type>/"`

## Naming

New files use `<timestamp>-<slug>.md` where timestamp is `date +%Y%m%d%H%M`.
Example: `202604031530-auth-redesign.md`. No type-specific prefixes. When
updating an existing blueprint, keep its current filename.

## Commit-on-Write

After every blueprint file write or move:

```sh
cd "$BLUEPRINTS_DIR" && \
  git add -A "<project>/" && \
  git commit -m "<type>(<project>): <slug>" && \
  git push || (git pull --rebase && git push)
```

- If the first push fails because the remote branch moved, pull with rebase
  and retry the push.
- If `git push` fails because no remote is configured, warn the user but do
  not treat it as a blocking error. The commit is still saved locally.
- If rebase fails, STOP and alert the user with conflict details. Do not
  continue — blueprint data may be at risk.

After a successful push, build and present a clickable link to the file on
the remote:

1. Derive the remote with
   `git -C "$BLUEPRINTS_DIR" remote get-url origin`.
2. Convert SSH or HTTPS remotes to a browser base:
   - `git@github.com:owner/repo.git` → `https://github.com/owner/repo`
   - `https://github.com/owner/repo.git` → `https://github.com/owner/repo`
3. Append `/blob/<branch>/<project>/<type>/<file>` using the branch that was
   pushed, normally `main`.
4. Include the resulting Markdown link in the user-facing response.
5. If the blueprint is a plan or design that precedes implementation, stop
   after presenting the link and wait for explicit user approval before
   making code changes.

## Plan Workflow

For non-trivial tasks, the plan file must be **written, committed, and
pushed as soon as the plan is generated** — *before* it is presented to
the user for approval. Do not gate the commit on user approval.

1. Research the codebase with read-only tools.
2. Draft the plan content.
3. Run the reuse-before-create check. If a matching plan exists, read it in
   full and integrate the new plan content into that file. Otherwise write a
   new plan to `$BLUEPRINTS_DIR/<project>/plan/<timestamp>-<slug>.md` using
   this structure:
   ```markdown
   # Plan: <Title>

   ## Summary

   - The intended outcome.
   - The chosen approach and major implementation phases.
   - Important constraints, risks, or approval state.

   **Date**: <YYYY-MM-DD>
   **Status**: Proposed

   ## Goal
   What we're trying to accomplish.

   ## Approach
   How we'll do it — key decisions, files affected, patterns used.

   ## Tasks
   Ordered list of implementation steps.
   ```
4. Immediately run commit-on-write to push the plan; capture the remote URL.
5. Present the plan summary and the URL to the user for approval.
6. Begin implementation only after the user approves.

## Archive Protocol

Archive relevant active blueprints when the user explicitly declares the
project or a workstream complete, or after the agent successfully pushes the
finishing implementation commits to the project repository's remote. Also
support direct requests to archive a named artifact.

A push is a completion signal only when the surrounding workflow establishes
that it finishes the approved work. Do not treat blueprint-repository pushes,
planning-only pushes, failed pushes, or intermediate implementation pushes as
completion.

Use the `archive-blueprint` skill to resolve the relevant set, protect unrelated
active efforts, preflight destination collisions and repository state, move the
set as one batch, and run commit-on-write once for the batch.

## Runtime-Specific Plan Mode Notes

- **Claude Code**: do not use the built-in `EnterPlanMode` tool for the
  design/planning phase — it blocks the Write tool, which would force the
  plan commit to happen only after approval via `ExitPlanMode`.
  Self-enforce read-only research instead, write and commit the plan, then
  present it (optionally via `ExitPlanMode` as a formal approval gate — the
  file is already committed at that point).
- **Codex**: if Plan Mode is active, do not write blueprint files during the
  planning turn. Produce the plan in chat. Once the user switches out of
  Plan Mode and asks for implementation or persistence, write, commit, and
  push the blueprint.
