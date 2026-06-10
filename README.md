# chifu dep-guard skill

An [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) for AI
coding agents (Claude Code, Cursor, and any Agent-Skills-compatible client) that
keeps your dependencies free of known vulnerabilities **in the coding loop** —
not in a separate Dependabot PR you deal with weeks later.

When your agent touches dependencies, this skill has it run the
[`chifu`](../chifu-cli) CLI to find vulnerable packages and **report them to
you** — the recommended upgrade for each, most-severe first — so you stay aware
of what entered your tree. The agent never edits dependency versions or runs
installs itself; detecting and surfacing the risk is its job, and applying the
fix in your own environment is yours.

## Requirements

This skill drives the **`chifu` CLI** — install it first (it works with no
account; a key just syncs results to your dashboard):

```bash
bunx @marshell/chifu@latest check          # zero-install, run in any project
# or install globally:
npm i -g @marshell/chifu@latest
```

## Install

### One command (recommended)

The [`chifu-wizard`](../chifu-wizard) installs the CLI and drops this skill into
the right place for your agent automatically:

```bash
bunx @marshell/chifu-wizard
```

### Manual — Claude Code

Copy `SKILL.md` to a folder named after the skill under `~/.claude/skills/`
(user-wide) or `.claude/skills/` (per-project):

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/chifu-dep-guard
cp SKILL.md ~/.claude/skills/chifu-dep-guard/SKILL.md
```

```powershell
# Windows (PowerShell)
New-Item -ItemType Directory -Force "$HOME\.claude\skills\chifu-dep-guard"
Copy-Item SKILL.md "$HOME\.claude\skills\chifu-dep-guard\SKILL.md"
```

The folder name (`chifu-dep-guard`) must match the `name:` in the frontmatter.
Claude Code picks up the skill on the next session.

### Manual — Cursor

Cursor reads project rules from `.cursor/rules/`. Copy `SKILL.md` in as a rule
file:

```bash
# macOS / Linux
mkdir -p .cursor/rules
cp SKILL.md .cursor/rules/chifu-dep-guard.mdc
```

```powershell
# Windows (PowerShell)
New-Item -ItemType Directory -Force ".cursor\rules"
Copy-Item SKILL.md ".cursor\rules\chifu-dep-guard.mdc"
```

The YAML frontmatter (`name`, `description`) doubles as Cursor rule metadata, so
the same file works in both clients.

## What it does

The agent runs `chifu check --json`, reads the rolled-up `packages` list, and
reports each vulnerable dependency — its installed version, the
`recommendedVersion` that clears its CVEs, the severity, and the per-CVE
`advisoryUrl` from `findings` — most-severe first. It never edits versions,
lockfiles, or `overrides`, and never runs an install. You apply the recommended
change in your own environment and can re-run the skill to confirm it's clean.

## License

MIT
