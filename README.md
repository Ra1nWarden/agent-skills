# Agent Skills

Portable [Agent Skills](https://agentskills.io) packages shared across AI
coding agents (Claude Code, Codex, and other runtimes that read
`SKILL.md`-based skill directories).

## Layout

```
skills/<name>/SKILL.md    # one skill per directory; name matches frontmatter
scripts/validate-skills   # frontmatter + secret-pattern checks
```

Each directory under `skills/` is a self-contained skill package. Frontmatter
uses the cross-agent fields `name` and `description`; some skills also carry
`argument-hint` and `allowed-tools`, which Claude Code honors and other
runtimes ignore.

## Skills

Blueprint lifecycle (persistent design artifacts in `$BLUEPRINTS_DIR`):

- `blueprints` — layout, naming, and commit conventions the others build on
- `research` — codebase research briefs
- `design` — design docs with adversarial review
- `implement` — approved design → dependency graph → waves → conformance review
- `review` — architecture / readability / correctness review passes
- `archive-blueprint` — move consumed blueprints to `archive/`

Work integrations:

- `assign-pr-reviews` — distribute PR review assignments
- `check-in` — engineer progress summaries from Asana/GitHub/Slack
- `file-asana-ticket` — synthesize Asana tickets from FigJam/Slack sources
- `write-asana-project-report` — Asana project status reports

## Consumption

The [dotfiles repo](https://github.com/Ra1nWarden/dotfiles) includes this
repository as a pinned git submodule and symlinks `skills/` to both
`~/.agents/skills` (Codex) and `~/.claude/skills` (Claude Code). Other hosts
can clone this repo directly and mount `skills/` read-only.

## Validation

```sh
./scripts/validate-skills
```

Checks that every skill directory has a `SKILL.md` with matching `name` and a
`description`, with no duplicate names and no obvious secret patterns.

## Provenance

Initial skills were migrated from `claude/commands/` and `codex/skills/` in
the dotfiles repo (as of dotfiles commit `1ff0335`), merging each duplicated
Claude-command / Codex-skill pair into one portable package.
