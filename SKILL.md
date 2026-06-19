---
slug: chifu-dep-guard
name: chifu dep-guard
homepage: https://marshell.dev
description: >-
  Dependency security gate for AI coding agents. Run chifu check --json before
  EVERY git commit, git push, pull request, npm install, bun install,
  package.json edit, or /bcp (build-chifu-push). Scan the lockfile for CVEs,
  fix by editing package.json (overrides for transitive deps), loop until clean.
  Supply chain, npm audit, CVE scanner, chifu CLI.
---

# chifu dep-guard

`chifu` scans your dependency tree against known CVEs. **You** fix what it
finds by editing `package.json` — never by installing packages.

---

# CRITICAL — run before commit, push, or PR

**NEVER `git commit`, `git push`, or open a PR until `vulnerablePackages` is 0**
(or only unfixable `recommendedVersion: null` entries remain and the user was
told).

1. **`cd` to the directory with `package.json`** (often `frontend/`, not git root)
2. **Run the check** (see [Preflight](#preflight) if anything fails):

   ```bash
   chifu check --json
   ```

   Fallback: `bunx @marshell/chifu@latest check --json`

3. **If `vulnerablePackages > 0`** → [fix loop](#fix-loop) until clean
4. **Only then** commit, push, or finish the task

Also run before: editing `package.json` / lockfile, adding or upgrading deps,
auditing supply chain, or when the user invokes **`/bcp`**.

---

## When to run

| Trigger | Action |
|---------|--------|
| User says commit / push / PR / merge | Check first; fix loop if needed |
| User invokes **`/bcp`** | Chifu is step 2 — after build, before commit (see [BCP workflow](#bcp-build-chifu-push)) |
| User says install / add / upgrade / pin a dep | Check after editing manifest; never install before check |
| Task touched `package.json` or lockfile | Check before ending the turn |
| User asks audit / CVE / supply chain / npm audit | Check + report + fix loop |
| No `package.json` in tree | Skip; tell the user |

---

## BCP — Build, Chifu, Push

When the user invokes **`/bcp`** (or says "bcp", "build check commit push"), run
the full gate **in order**. Do not commit or push unless **both** build and
chifu pass.

### 1. Find the package root

Run from the directory with `package.json` and a `"build"` script:

1. Directory the user named explicitly
2. Nearest ancestor of cwd with `package.json` + `"build"`
3. Ask if ambiguous (monorepos)

### 2. Build

```bash
bun run build
```

- Fail → fix, re-run build, **stop** (no chifu, commit, or push)
- Generated files in diff → stage only intentional changes at commit time

### 3. Chifu check (this skill)

```bash
chifu check --json
```

Follow the [fix loop](#fix-loop) until `vulnerablePackages` is 0. After fixing
deps, **re-run `bun run build`** before continuing.

### 4. Commit (git root)

Only after build + chifu pass. From the **git root**:

```bash
git status && git diff && git log -5 --oneline
```

Stage relevant files (never `.env` or secrets). Skip commit if clean tree.
Draft a concise message (focus on **why**). `/bcp` authorizes commit even when
the user would not normally ask.

### 5. Push

```bash
git push
```

Use `-u origin HEAD` when no upstream. Never force-push main/master unless
explicitly requested.

### 6. Report

Build · Chifu (fixes applied?) · Commit hash or skipped · Push result.

---

## Hard rules

### Never run an install (except lockfile-only)

`chifu check` reads `package.json` and the lockfile from disk — **nothing needs
to be installed**. Never run:

`npm install` · `npm ci` · `npm audit fix` · `yarn` · `pnpm install` ·
`bun install` · `bun update`

Install scripts run arbitrary code from packages you are vetting — the exact
attack chifu exists to catch.

**One exception** after editing `package.json`:

```bash
npm install --package-lock-only --ignore-scripts
```

Rewrites the lockfile without downloading or executing packages.

### Use `recommendedVersion` verbatim

Never invent or round a version. A fake version (e.g. `lodash@4.18.0`) can
resolve to a typosquat. If `recommendedVersion` is `null`, report CVEs and
suggest mitigations — do not guess. Never remove a dependency without asking.

### Loop until clean

Never end a turn with a fixable vulnerability open. Never claim a dependency
is safe without a fresh `chifu check --json`.

---

## Fix loop

Drive this until `vulnerablePackages` is 0:

```
chifu check --json
  → edit package.json (versions + overrides/resolutions)
  → npm install --package-lock-only --ignore-scripts
  → chifu check --json
  → repeat
```

One bump can surface new transitive CVEs — keep looping.

### Fix recipes

**Direct dependency** — set version in `dependencies` / `devDependencies`:

```json
"lodash": "4.17.21"
```

**Transitive (npm / bun)** — add or update `overrides`:

```json
"overrides": { "lodash": "4.17.21" }
```

**Transitive (yarn)** — add or update `resolutions`:

```json
"resolutions": { "lodash": "4.17.21" }
```

**Breaking major bump** — read `advisoryUrl` and changelog; fix call sites by
static review. Do not install to test.

**No fix yet** (`recommendedVersion: null`) — report severity, CVEs, and
`advisoryUrl`; suggest override away from the vulnerable range or dropping the
dep (with user approval).

---

## Procedure

### 1. Preflight

| Symptom | Action |
|---------|--------|
| `chifu: command not found` | `bunx @marshell/chifu@latest check --json` |
| `you're not signed in` | `chifu login` or set `CHIFU_API_KEY=chf_…` — **stop** |
| `no package.json` | Wrong directory — find the package root |
| `check failed` (network/API) | Report error — never invent CVEs |
| Exit code **2** | Error — do not commit |

Exit codes: **0** = ok (use `--fail-on-findings` in CI for non-zero on vulns),
**1** = vulns found (with `--fail-on-findings`), **2** = error.

### 2. Parse JSON output

`packages` = actionable rollup; `findings` = per-CVE detail.

```json
{
  "vulnerablePackages": 1,
  "scanned": 905,
  "packages": [
    {
      "name": "lodash",
      "version": "4.17.4",
      "recommendedVersion": "4.17.21",
      "worstSeverity": "high",
      "cveCount": 5,
      "cves": ["CVE-2021-23337"]
    }
  ],
  "findings": [
    {
      "name": "lodash",
      "version": "4.17.4",
      "cve": "CVE-2021-23337",
      "severity": "high",
      "vulnerableRange": "<4.17.21",
      "fixedVersion": "4.17.21",
      "advisoryUrl": "https://github.com/advisories/...",
      "summary": "..."
    }
  ],
  "update": null
}
```

If `update` is present (e.g. `{ "status": "updated", "from": "0.2.0", "to":
"0.3.0" }`), mention it in one line. If `vulnerablePackages` is 0, say so and
stop (unless `/bcp` still needs commit/push).

Fix most-severe packages first.

### 3. Report when done

Tell the user: packages bumped (from → to), overrides added, breaking-change
notes. Remind them to run full `npm install` + build + tests in **their**
environment to apply and validate.

---

## Monorepos

- Run from **each** directory with its own `package.json` + lockfile before
  committing that subtree.
- User working in `frontend/` → check there, not the git root.
- Multiple packages changed → check each one.

---

## CLI reference

```bash
chifu check [path] --json       # machine output for this skill
chifu check --verbose           # list every CVE
chifu check --fail-on-findings  # exit 1 when vulns found (CI)
chifu login                     # browser pairing
chifu login <chf_key>           # save key (CI)
```

**Environment:** `CHIFU_API_KEY` · `CHIFU_API_URL` · `CHIFU_WEB_URL` ·
`CHIFU_NO_UPDATE=1`

---

## Done checklist

- [ ] `vulnerablePackages === 0` (or unfixable entries reported to user)
- [ ] User told: bumps, overrides, breaking-change notes
- [ ] Under `/bcp`: build passed, then chifu, then commit/push if requested
- [ ] Never claimed "safe" without a fresh `chifu check --json`
