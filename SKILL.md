---
slug: chifu-dep-guard
name: chifu dep-guard
homepage: https://marshell.dev
description: >-
  Dependency security gate for AI coding agents. Run chifu check --json before
  EVERY git commit, git push, pull request, dependency manifest edit, package
  manager install, or /bcp (build-chifu-push). Scans lockfiles/manifests for
  CVEs across npm, PyPI, Maven, Go, NuGet, crates.io, Packagist, RubyGems.
  Fix by editing the ecosystem manifest (overrides for npm transitives), loop
  until clean. Supply chain, CVE scanner, chifu CLI.
---

# chifu dep-guard

`chifu` scans your dependency tree against known CVEs. **You** fix what it
finds by editing the project's dependency manifest — never by installing
packages (except the npm lockfile-only exception below).

---

# CRITICAL — run before commit, push, or PR

**NEVER `git commit`, `git push`, or open a PR until `vulnerablePackages` is 0**
(or only unfixable `recommendedVersion: null` entries remain and the user was
told).

1. **`cd` to the project root** with a supported manifest/lockfile (often
   `frontend/`, not git root)
2. **Run the check** (see [Preflight](#preflight) if anything fails):

   ```bash
   chifu check --json
   ```

   Fallback: `bunx @marshell/chifu@latest check --json`

3. Read `ecosystem` in the JSON — it tells you which [fix recipe](#fix-recipes)
   applies
4. **If `vulnerablePackages > 0`** → [fix loop](#fix-loop) until clean
5. **Only then** commit, push, or finish the task

Also run before: editing dependency manifests or lockfiles, adding or upgrading
deps, auditing supply chain, or when the user invokes **`/bcp`**.

---

## Supported ecosystems

`chifu check` auto-detects one ecosystem per run (first detected manifest with
resolved deps). Supported:

| Ecosystem | Manifests / lockfiles |
|-----------|------------------------|
| **npm** | `package-lock.json`, `npm-shrinkwrap.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lock`, `package.json`, `node_modules` |
| **PyPI** | `requirements.txt`, `poetry.lock`, `Pipfile.lock`, `uv.lock`, `pylock.toml`, `pyproject.toml` |
| **Maven** | `pom.xml`, `build.gradle`, `build.gradle.kts`, Gradle lockfiles |
| **Go** | `go.mod`, `go.sum` |
| **NuGet** | `packages.lock.json`, `packages.config`, `.csproj` |
| **crates.io** | `Cargo.lock`, `Cargo.toml` |
| **Packagist** | `composer.lock`, `composer.json` |
| **RubyGems** | `Gemfile.lock`, `gems.locked`, `Gemfile` |

If multiple ecosystems exist in one directory, chifu picks the first with
resolved dependencies (npm → PyPI → Maven → Go → NuGet → crates.io → Packagist →
RubyGems). Monorepos with mixed stacks: run `chifu check` from each subproject.

---

## When to run

| Trigger | Action |
|---------|--------|
| User says commit / push / PR / merge | Check first; fix loop if needed |
| User invokes **`/bcp`** | Chifu is step 2 — after build, before commit (see [BCP workflow](#bcp-build-chifu-push)) |
| User says install / add / upgrade / pin a dep | Check after editing manifest; never install before check |
| Task touched a manifest or lockfile | Check before ending the turn |
| User asks audit / CVE / supply chain | Check + report + fix loop |
| No supported manifest in tree | Skip; tell the user |

---

## BCP — Build, Chifu, Push

When the user invokes **`/bcp`** (or says "bcp", "build check commit push"), run
the full gate **in order**. Do not commit or push unless **both** build and
chifu pass.

### 1. Find the package root

Run from the directory with a supported manifest and a project build step:

1. Directory the user named explicitly
2. Nearest ancestor with a supported manifest + build script (`package.json`
   `"build"`, `Makefile`, `go build`, `cargo build`, etc.)
3. Ask if ambiguous (monorepos)

### 2. Build

Use the project's normal build command (`bun run build`, `npm run build`,
`go build ./...`, `cargo build`, `mvn package`, etc.).

- Fail → fix, re-run build, **stop** (no chifu, commit, or push)
- Generated files in diff → stage only intentional changes at commit time

### 3. Chifu check (this skill)

```bash
chifu check --json
```

Follow the [fix loop](#fix-loop) until `vulnerablePackages` is 0. After fixing
deps, **re-run the project build** before continuing.

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

Build · Chifu (ecosystem, fixes applied?) · Commit hash or skipped · Push result.

---

## Hard rules

### Never run an install (except npm lockfile-only)

`chifu check` reads manifests and lockfiles from disk — **nothing needs to be
installed** to scan. Never run:

`npm install` · `npm ci` · `npm audit fix` · `yarn` · `pnpm install` ·
`bun install` · `bun update` · `poetry install` · `pip install` ·
`go mod download` · `cargo fetch` · `composer install` · `bundle install`

Install/fetch commands run arbitrary code or pull unaudited artifacts — the
exact attack chifu exists to catch.

**One exception** after editing **npm** `package.json`:

```bash
npm install --package-lock-only --ignore-scripts
```

Rewrites `package-lock.json` without downloading or executing packages.

For other ecosystems: edit the manifest **and** lockfile directly when the
format is straightforward; otherwise tell the user to regenerate the lockfile
in their environment after your manifest edits.

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
  → read ecosystem + packages[]
  → edit manifest (and lockfile) per ecosystem recipe
  → npm only: npm install --package-lock-only --ignore-scripts
  → chifu check --json
  → repeat
```

One bump can surface new transitive CVEs — keep looping.

### Fix recipes

Use `recommendedVersion` from `packages[]` for every bump.

#### npm

**Direct** — bump in `dependencies` / `devDependencies`:

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

Then: `npm install --package-lock-only --ignore-scripts`

#### PyPI

Bump pinned versions in whichever file chifu scanned (`requirements.txt`,
`pyproject.toml`, `poetry.lock`, `uv.lock`, etc.). For lockfile-only formats,
edit the locked version entry directly when possible.

#### Maven / Gradle

Bump `<version>` in `pom.xml`, or the version constraint in `build.gradle` /
`build.gradle.kts`. Update Gradle lockfile entries when present.

#### Go

Bump the module version in `go.mod` `require` directives. Update matching
`go.sum` entries when you can do so safely; otherwise note that the user should
run `go mod tidy` locally.

#### NuGet

Bump `PackageReference` `Version` in `.csproj` or entries in
`packages.lock.json` / `packages.config`.

#### crates.io

Bump version in `Cargo.toml` `[dependencies]`. Update `Cargo.lock` package
version entries when editing the lockfile directly is feasible.

#### Packagist

Bump version in `composer.json` `require` / `require-dev`. Update
`composer.lock` hash blocks when editing directly is feasible.

#### RubyGems

Bump version in `Gemfile`. Update `Gemfile.lock` / `gems.locked` entries when
feasible.

#### All ecosystems

**Breaking major bump** — read `advisoryUrl` and changelog; fix call sites by
static review. Do not install to test.

**No fix yet** (`recommendedVersion: null`) — report severity, CVEs, and
`advisoryUrl`; suggest pinning away from the vulnerable range or dropping the
dep (with user approval).

---

## Procedure

### 1. Preflight

| Symptom | Action |
|---------|--------|
| `chifu: command not found` | `bunx @marshell/chifu@latest check --json` |
| `you're not signed in` | `chifu login` or set `CHIFU_API_KEY=chf_…` — **stop** |
| `no supported package manifest` | Wrong directory — find the project root |
| `check failed` (network/API) | Report error — never invent CVEs |
| Exit code **2** | Error — do not commit |

Exit codes: **0** = ok (use `--fail-on-findings` in CI for non-zero on vulns),
**1** = vulns found (with `--fail-on-findings`), **2** = error.

### 2. Parse JSON output

`ecosystem` = which fix recipe to use. `packages` = actionable rollup;
`findings` = per-CVE detail.

```json
{
  "ecosystem": "npm",
  "repo": "org/repo",
  "scanned": 905,
  "source": "package-lock.json (full tree)",
  "vulnerablePackages": 1,
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

Tell the user: ecosystem, packages bumped (from → to), overrides/constraints
added, breaking-change notes. Remind them to run their normal install + build
+ tests in **their** environment to apply and validate.

---

## Monorepos

- Run from **each** subproject with its own manifest + lockfile before
  committing that subtree.
- User working in `frontend/` → check there, not the git root.
- Multiple packages or ecosystems changed → check each one separately.
- Mixed-ecosystem folders: chifu scans only one ecosystem per invocation —
  `cd` into the subproject or accept the auto-selected one.

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
- [ ] User told: ecosystem, bumps, overrides/constraints, breaking-change notes
- [ ] Under `/bcp`: build passed, then chifu, then commit/push if requested
- [ ] Never claimed "safe" without a fresh `chifu check --json`
