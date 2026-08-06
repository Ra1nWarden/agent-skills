---
name: review
description: Use when the user asks for a code review of a commit, branch, pull request, or current diff, especially with specialized architecture, readability, and correctness review passes.
argument-hint: <commit hash, PR number/URL, or branch name; omit to review the current diff>
allowed-tools: Read Glob Grep Bash(ls *) Bash(cat *) Bash(git log*) Bash(git diff*) Bash(git status*) Bash(git show*) Bash(git branch*) Bash(git rev-parse*) Bash(git add*) Bash(git commit*) Bash(git push*) Bash(git pull*) Bash(pwd) Bash(tree *) Bash(gh pr view*) Bash(gh pr diff*) Bash(date *) Bash(basename *) Bash(cd *) Bash(mkdir *)
---

# Code Review Workflow

You are reviewing a code change using three specialized review passes. Use a
code-review stance: findings first, ordered by severity, with file and line
references.

## Phase 0: Environment Setup

Follow the `blueprints` skill: check `$BLUEPRINTS_DIR` and derive `$PROJECT`.
If `$BLUEPRINTS_DIR` is empty, reviews will not be persisted — warn the user
but continue; review output is still presented inline.

---

**Input**: The user provides one of:
- A **commit hash** (e.g., `abc1234`)
- A **PR number or URL** (e.g., `#42` or `https://github.com/org/repo/pull/42`)
- A **branch name** to compare against the main branch
- **Nothing** — review the current uncommitted diff

---

## Phase 1: Gather the Changes

Based on the input type:

- **Commit hash**: Run `git show --stat <hash>` to get files changed, then
  `git show <hash>` for the full diff.
- **PR**: Run `gh pr view <number> --json title,body,baseRefName,headRefName` for
  context, then `gh pr diff <number>` for the full diff.
- **Branch name**: Run `git diff main...<branch> --stat` for files changed,
  then `git diff main...<branch>` for the full diff.
- **No target**: Run `git status` and `git diff` (plus `git diff --staged`)
  for the current changes.

Collect:
1. The list of all files changed
2. A summary of what the change is trying to accomplish (from commit message,
   PR description, or diff context)
3. The full diff content

---

## Phase 2: Parallel Review

Run three focused review passes. If specialized reviewer agents are available,
**spawn them in parallel**, passing each one the list of files changed, the
change summary, and the full diff; if not, perform the passes yourself, one at
a time:

1. **`architect-review`** — structural problems, component boundaries,
   dependency hygiene, API shape
2. **`readability-review`** — language best practices, style guide conformance,
   naming, clarity
3. **`correctness-review`** — edge cases, unhappy paths, logic errors, missing
   unit tests, security boundaries

Wait for all three to complete.

---

## Phase 3: Present Results

Present the reviews to the user in a structured format:

```
## Review: <change description>

### Architecture
<architect-review verdict and issues>

### Readability
<readability-review verdict and issues>

### Correctness
<correctness-review verdict and issues>

---

### Overall
- Critical issues: <count>
- Warnings: <count>
- Nits: <count>
- Overall verdict: PASS / PASS WITH WARNINGS / FAIL
```

The overall verdict is:
- **FAIL** if any reviewer returned FAIL
- **PASS WITH WARNINGS** if any reviewer returned PASS WITH WARNINGS
- **PASS** only if all three reviewers returned PASS

Findings come first; include open questions or assumptions, and give a brief
change summary only after findings.

---

## Phase 4: Persist to Blueprints (if available)

If `$BLUEPRINTS_DIR` is set:

1. Save the full review output to
   `$BLUEPRINTS_DIR/$PROJECT/review/$(date +%Y%m%d%H%M)-<slug>.md`
   where `<slug>` is derived from the change description.
2. **Commit-on-write**: Run the `blueprints` commit protocol with message
   `review($PROJECT): <slug>`.
3. Tell the user where the review was saved.
