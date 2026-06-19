# chifu dep-guard skill

An [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) for AI
coding agents (Claude Code, Cursor, and any Agent-Skills-compatible client) that
keeps your dependencies free of known CVEs **in the coding loop** — before you
commit, push, or ship.

The agent runs [`chifu`](../chifu-cli), reads `vulnerablePackages` from
`chifu check --json`, fixes issues by editing `package.json` (and lockfile-only
re-resolve), and loops until clean. It also defines the **chifu step** inside
**`/bcp`** (Build → Chifu → Push).

## What it does

- **Gate** every `git commit`, `git push`, PR, and dependency change
- **Fix loop** — bump direct deps, add `overrides` / `resolutions` for transitives
- **`/bcp`** — after `bun run build`, run chifu; only then commit and push
- **Never install** except `npm install --package-lock-only --ignore-scripts`

See [SKILL.md](./SKILL.md) for the full agent workflow.

## Requirements

```bash
bunx @marshell/chifu@latest check --json   # zero-install
# or:
npm i -g @marshell/chifu@latest
chifu login
```

## Install

### skills.sh (recommended)

```bash
npx skills add marshell-dev/chifu-skill --global --yes
npx skills update chifu-dep-guard --global --yes   # refresh later
```

### chifu wizard

```bash
bunx @marshell/chifu-wizard
```

### Manual — Claude Code

```bash
mkdir -p ~/.claude/skills/chifu-dep-guard
cp SKILL.md ~/.claude/skills/chifu-dep-guard/SKILL.md
```

### Manual — Cursor

```bash
mkdir -p .cursor/rules
cp SKILL.md .cursor/rules/chifu-dep-guard.mdc
```

## License

MIT
