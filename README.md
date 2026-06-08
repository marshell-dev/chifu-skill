# chifu dep-guard skill

An [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) for AI
coding agents (Claude Code, Cursor, and any Agent-Skills-compatible client) that
keeps your dependencies free of known vulnerabilities **in the coding loop** —
not in a separate Dependabot PR you deal with weeks later.

When your agent touches dependencies, this skill has it run the
[`chifu`](../chifu-cli) CLI to find vulnerable packages, then upgrade them and
fix the resulting breaking changes before the work is considered done. The CLI
only detects; the agent fixes.

## Requirements

This skill drives the **`chifu` CLI** — install it first (it works with no
account; a key just syncs results to your dashboard):

```bash
bunx @mfinikov/chifu check          # zero-install, run in any project
# or install globally:
npm i -g @mfinikov/chifu
```

## Install

### One command (recommended)

The [`chifu-wizard`](../chifu-wizard) installs the CLI and drops this skill into
the right place for your agent automatically:

```bash
bunx @mfinikov/chifu-wizard
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

The agent runs `chifu check --json`, reads the rolled-up `packages` list,
upgrades each vulnerable dependency to its `recommendedVersion`, fixes any
breaking changes using the per-CVE `advisoryUrl` in `findings`, and re-runs
until clean.

## License

MIT
