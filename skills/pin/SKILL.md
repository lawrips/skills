---
name: pin
description: Baseline supply-chain hygiene audit. Pins direct package versions, ensures lockfiles are committed, flags secret files, and audits build scripts for safer package-manager usage. Use when setting up a new project, onboarding a repo, or after a dependency scare.
disable-model-invocation: true
compatibility: Node.js / Bun / Python projects. Claude Code only.
metadata:
  author: lawrips
  version: 1.5.0
---

# Pin

You are performing a baseline supply-chain hygiene audit. Work through each check in order, report findings as you go, and offer to fix each issue. Do not read the contents of any file that may contain secrets.

## DISCLAIMER

This skill is a **baseline supply-chain hygiene audit**, not a comprehensive security review.

- Do **NOT** tell the operator that the project is "secure," "safe," or "locked down" just because this skill passes.
- A clean result means the repo is better pinned, more consistent, and less likely to drift or execute undeclared tooling by accident.
- A clean result does **NOT** mean the dependency graph is trustworthy, vulnerability-free, or safe from compromise.
- This skill does **NOT** evaluate transitive dependencies, maintainer trust, package provenance, signed releases, registry compromise, CI isolation, or whether a package is malicious.
- This skill does **NOT** replace deeper follow-up work such as vulnerability scanning, package trust review, or a manual dependency audit.
- Treat every recommendation as a proposed hygiene fix, not a guarantee.

## Principles

- **Scan, don't read secrets.** Use `ls`, `find`, and `git ls-files` to detect secret files. Never `cat` or `Read` their contents.
- **Report before fixing.** Present all findings first, then offer to fix. Don't silently change anything.
- **Respect the project's package manager.** Ask the operator which package manager the project uses and tailor all recommendations accordingly.
- **Load only the relevant ecosystem reference.** After Step 1, use `references/node.md` for Node/Bun projects and `references/python.md` for Python projects. If both ecosystems are present, use both.

---

## Step 1: Identify Ecosystem and Package Manager

This is a 3-phase gate. Complete all three before proceeding to Step 2.

### 1a — Scan all signals

Scan the entire project, not just root, for every package-manager signal.

**Node/Bun lockfiles:**
- `package-lock.json` → npm
- `bun.lockb` OR `bun.lock` → bun
- `pnpm-lock.yaml` → pnpm
- `yarn.lock` → yarn

**Python lockfiles and manifests:**
- `uv.lock` → uv
- `poetry.lock` → Poetry
- `pdm.lock` → PDM
- `Pipfile.lock` → Pipenv
- `requirements*.txt` / `constraints*.txt` → pip or pip-tools
- `pyproject.toml`, `setup.cfg`, `setup.py`, `requirements*.in`, `Pipfile` → Python manifest signals

**Build and deploy scripts** — scan these locations for package-manager commands:
- `package.json` scripts, including nested package.json files
- `pyproject.toml` tool scripts, where present
- `Dockerfile` / `Dockerfile.*` / `docker-compose.yml`
- `Makefile` / `Justfile`
- `.github/workflows/*.yml` / `.gitlab-ci.yml`
- Any `deploy.sh` or similar scripts

Scan for Node/Bun commands: `npm`, `bun`, `npx`, `bunx`, `pnpm`, `yarn`.

Scan for Python commands: `uv`, `uvx`, `pipx`, `python -m pip`, `pip`, `pip3`, `pip-compile`, `pip-sync`, `poetry`, `pdm`, `pipenv`, `hatch`, `tox`, `nox`.

Runner commands are package-manager signals too. A project with `bunx` in a deploy script is using bun regardless of what lockfiles exist. A Python project with `uvx ruff` or `pipx run` is executing tooling outside the project environment unless that tool is otherwise declared and locked.

### 1b — Report findings

Present a single table showing every signal and where it came from:

| Signal | Ecosystem | Package Manager | Location |
|--------|-----------|-----------------|----------|
| package-lock.json | Node | npm | root, mobile-app/ |
| bun.lock | Node | bun | root |
| `npm ci` | Node | npm | Dockerfile.dev:13 |
| `bunx wrangler` | Node | bun | package.json deploy script |
| uv.lock | Python | uv | root |
| `uv sync` | Python | uv | .github/workflows/test.yml:24 |
| poetry.lock | Python | Poetry | api/ |

If only one package manager is detected in each ecosystem, report it and move on. If mixed usage is detected within an ecosystem, proceed to 1c.

After Step 1b, load the relevant reference:
- Node/Bun signals → `references/node.md`
- Python signals → `references/python.md`

### 1c — Standardize (if mixed)

If multiple package managers are detected within the same ecosystem, stop and ask the operator:

1. **Standardize on one** — which one? The skill will flag all commands using the wrong PM and offer to migrate them.
2. **Keep mixed** — the skill audits each detected PM but notes the risk (Claude may use the wrong one, lockfiles may drift out of sync).

Wait for the answer before proceeding. The chosen PM determines the correct commands for Steps 2–6.

If the operator chooses to standardize, flag the wrong PM's lockfile for removal and all wrong-PM commands for migration as findings in Steps 3 and 4.

---

## Step 2: Audit Package Versions

Use the relevant ecosystem reference to identify manifest files and version syntax.

Scan every dependency manifest in the project for unpinned direct dependencies. Include production dependencies, development dependencies, optional dependencies, and build-system requirements where the ecosystem supports them.

Present findings as a table:

| Location | Package | Current | Issue |
|----------|---------|---------|-------|
| package.json | axios | ^1.14.0 | Range — pins to 1.14.0 |
| pyproject.toml | pydantic | >=2.0 | Range — pins to 2.13.3 |

To determine the correct pinned version, check the chosen PM's lockfile for the currently resolved version. If the lockfile doesn't exist yet, use the fallback lookup described in the relevant ecosystem reference. Offer to fix all at once.

Do not execute Python `setup.py` files while auditing. Treat them as source text only.

---

## Step 3: Audit Lockfile

Use the relevant ecosystem reference to identify the chosen PM's lockfile.

Four checks for each chosen PM's lockfile:

1. **Lockfile exists** — does the lockfile for the chosen package manager exist?
2. **Lockfile is tracked by git** — run `git ls-files <lockfile>` to verify it's committed, not just present on disk
3. **Lockfile not gitignored** — use `git check-ignore -v <lockfile>` to verify whether the lockfile is actually ignored. Do not rely on scanning `.gitignore` text. If the lockfile is ignored, flag it as a FAIL — lockfiles must be committed
4. **Wrong-PM lockfiles** — if the operator chose to standardize in Step 1c, flag any lockfiles from the other PM for removal

Report pass/fail for each. Offer to fix `.gitignore` if the lockfile is ignored, and offer to delete wrong-PM lockfiles.

---

## Step 4: Audit Build Scripts

Use the chosen package manager from Step 1 and the relevant ecosystem reference to determine the correct commands.

Scan all build/deploy locations identified in Step 1a and flag three types of issues:

### Wrong package manager

If the operator chose to standardize in Step 1c, flag any **package management command** using the wrong PM — install commands, lockfile references, runner commands.

Runtime and container changes are a separate concern. Dockerfile base images, runtime CMD entries, Python interpreter installation, Node runtime installation, and similar changes are NOT package management hygiene. Present these as additional migration/policy changes in a separate section, not as required fixes.

### Unsafe install commands

Use the ecosystem reference to identify unsafe install commands. In general, flag commands that:
- Do a bare install instead of using the lockfile
- Re-resolve the dependency tree during CI/deploy
- Execute undeclared tooling outside the project lockfile
- Omit the ecosystem's available install hardening flags

Node package managers have `--ignore-scripts`; Python package managers do not have an exact equivalent. Do not invent an `--ignore-scripts` finding for Python. Use the Python reference for the Python-specific install risks to flag.

### Runner commands

Runner commands need special care. If the target tool is already declared in the relevant manifest and present in the lockfile, the safest approach is usually to rely on that declared dependency rather than introducing a separate version pin in the runner command.

If the target tool is not declared locally:
- Flag it for review.
- Preferred fix: add the package as a development dependency with an exact version, then invoke it through the project's normal scripts/tooling.
- Acceptable fallback: pin the exact package name and version in the runner command if adding a dependency is not appropriate and the runner supports exact pins.

If the binary name differs from the package name, verify the actual package name before recommending a change. Do not guess from the binary name alone.

Present findings and offer to fix.

### Project-wide PM hardening (recommended, not required)

Use the ecosystem reference to run the explicit hardening checks for the chosen PM. Report pass/fail and mark them as **recommended**, not required hygiene fixes.

---

## Step 5: Scan for Secret Files

Scan the filesystem for files that likely contain secrets. **Do not read their contents — detection is by filename only.**

Use `find` or Glob to locate files matching these patterns in the project root (exclude `node_modules`, Python virtualenvs, and cache directories):

```
.env, .env.*, .env.local, .env.production
.dev.vars
credentials*, *secret*, *token*
*.pem, *.key, *.p12
.vscode/launch.json
```

You MUST actually search the filesystem — do not just check `.gitignore` and assume. A file can exist on disk without being in `.gitignore`, or be in `.gitignore` but not in `permissions.deny`.

For each file found on disk, check TWO things:
1. **Is it ignored by git?** — use `git check-ignore -v <path>`. If not ignored, flag as FAIL
2. **Is it in `.claude/settings.local.json` permissions.deny?** — if not, flag as FAIL (needs Read/Write/Edit deny triple)

Present findings as a table:

| File | Ignored by git | In permissions.deny |
|------|----------------|-------------------|
| .env | ✓ | ✗ — needs adding |
| .dev.vars | ✓ | ✓ |

Offer to fix both git ignore rules (for example in `.gitignore`) and `.claude/settings.local.json`. When adding to `settings.local.json`, use the deny triple pattern:

```json
"Read(<path>)", "Write(<path>)", "Edit(<path>)"
```

---

## Step 6: Audit Project CLAUDE.md

Check if the project has a `CLAUDE.md` file at the project root. If it exists, scan for supply chain and safety reminders. If missing or incomplete, offer to add:

- Package manager preference (e.g. "This project uses bun. Never use npm." or "This project uses uv. Never use pip directly." — based on the operator's choice in Step 1)
- Supply chain safety (locked install commands, exact pins, install hardening, lockfile commitment)
- Git safety (no destructive commands, no force push, no chained push)
- Secret handling (don't read .env files, ask user for secret values)

Do not overwrite existing content — append a new section if the file already exists.

---

## Summary

After all checks, present a summary:

First, frame the result accurately:

```
This was a baseline supply-chain hygiene audit, not a comprehensive security review.
A clean result means the repo is better pinned, more consistent, and less likely to drift or execute undeclared tooling.
It does NOT mean the dependency graph is trustworthy, vulnerability-free, or safe from compromise.
```

Then present the summary:

```
Pin audit complete:
- Ecosystems / package managers: <detected choices>
- Package versions: X pinned, Y need fixing
- Lockfile: <status>
- Build scripts: X clean, Y need fixing
- PM hardening (recommended): <status from ecosystem reference>
- Additional migration changes: X may be needed to complete PM standardization in this repo
- Secret files: X protected, Y need fixing
- Project CLAUDE.md: <status>
```

Then ask whether the operator wants to apply:

- only the required hygiene fixes
- the required hygiene fixes plus any additional migration/policy changes
- one group at a time

After applying fixes, recommend the operator run the chosen PM's locked install command from the relevant ecosystem reference to verify the pinned versions resolve cleanly.

If the operator wants deeper assurance, suggest separate follow-up work such as vulnerability scanning or package trust review. Make clear that this is different from pinning and lockfile hygiene.

---

## Important

- **Never read secret file contents.** Detection is by filename only.
- **Always report before fixing.** The operator decides what to change.
- **Be specific about what changes.** Show the exact edit before making it.
- **This is a point-in-time audit.** Ongoing enforcement comes from hooks (bash-guard.sh, tkt-remind.sh) and permissions (settings.json). This skill sets up the project; hooks maintain it.
