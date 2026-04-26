---
name: pin
description: Baseline supply-chain hygiene audit. Pins direct package versions, ensures lockfiles are committed, flags secret files, and audits build scripts for safer package-manager usage. Use when setting up a new project, onboarding a repo, or after a dependency scare.
compatibility: Node.js / Bun projects. Codex only.
metadata:
  author: lawrips
  version: 1.4.0
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

- **Scan, don't read secrets.** Use `ls`, `find`, and `git ls-files` to detect secret files. Never `cat`, open, or read their contents.
- **Report before fixing.** Present all findings first, then offer to fix. Don't silently change anything.
- **Respect the project's package manager.** Ask the operator which package manager the project uses and tailor all recommendations accordingly.

---

## Step 1: Identify Package Manager

This is a 3-phase gate. Complete all three before proceeding to Step 2.

### 1a — Scan all signals

Scan the entire project (not just root) for every package manager signal:

**Lockfiles:**
- `package-lock.json` → npm
- `bun.lockb` OR `bun.lock` → bun (bun uses either format depending on version)
- `pnpm-lock.yaml` → pnpm
- `yarn.lock` → yarn

**Build and deploy scripts** — scan these locations for PM commands (`npm`, `bun`, `npx`, `bunx`, `pnpm`, `yarn`):
- `package.json` scripts (all of them, including nested package.json files)
- `Dockerfile` / `Dockerfile.*` / `docker-compose.yml`
- `Makefile` / `Justfile`
- `.github/workflows/*.yml` / `.gitlab-ci.yml`
- Any `deploy.sh` or similar scripts

**Runner commands** — `npx` and `bunx` are PM signals too. A project with `bunx` in its deploy script is using bun regardless of what lockfiles exist.

### 1b — Report findings

Present a single table showing every signal and where it came from:

| Signal | Package Manager | Location |
|--------|----------------|----------|
| package-lock.json | npm | root, mobile-app/ |
| bun.lock | bun | root |
| `npm ci` | npm | Dockerfile.dev:13 |
| `bunx opennextjs-cloudflare` | bun | package.json deploy script |
| `bunx wrangler` | bun | package.json deploy script |

If only one PM is detected, report it and move on. If mixed usage is detected, proceed to 1c.

### 1c — Standardize (if mixed)

If multiple package managers are detected, stop and ask the operator:

1. **Standardize on one** — which one? The skill will flag all commands using the wrong PM and offer to migrate them.
2. **Keep mixed** — the skill audits both but notes the risk (Codex may use the wrong one, lockfiles may drift out of sync).

Wait for the answer before proceeding. The chosen PM determines the correct commands for Steps 2–6.

If the operator chooses to standardize, flag the wrong PM's lockfile for removal and all wrong-PM commands for migration as findings in Steps 3 and 4.

---

## Step 2: Audit Package Versions

Scan every `package.json` in the project (root and nested packages/workspaces) for unpinned versions in `dependencies` and `devDependencies`. Flag any version that starts with `^`, `~`, `>=`, `>`, `*`, or `latest`.

Present findings as a table:

| Location | Package | Current | Issue |
|----------|---------|---------|-------|
| package.json | axios | ^1.14.0 | Range — pins to 1.14.0 |
| apps/web/package.json | lodash | ~4.17.0 | Range — pins to 4.17.0 |

To determine the correct pinned version, check the lockfile for the currently resolved version. If the lockfile doesn't exist yet (new project), fall back to `npm view <pkg> version` to get the current latest (this works regardless of which PM the project uses). Offer to fix all at once.

---

## Step 3: Audit Lockfile

Four checks for the chosen PM's lockfile:

1. **Lockfile exists** — does the lockfile for the chosen package manager exist? For bun, check both `bun.lock` and `bun.lockb` — if both exist, flag the old format (`bun.lockb`) for removal.
2. **Lockfile is tracked by git** — run `git ls-files <lockfile>` to verify it's committed, not just present on disk
3. **Lockfile not gitignored** — use `git check-ignore -v <lockfile>` to verify whether the lockfile is actually ignored. Do not rely on scanning `.gitignore` text. If the lockfile is ignored, flag it as a FAIL — lockfiles must be committed
4. **Wrong-PM lockfiles** — if the operator chose to standardize in Step 1c, flag any lockfiles from the other PM for removal (e.g. project chose bun → flag `package-lock.json` for deletion)

Report pass/fail for each. Offer to fix .gitignore if the lockfile is ignored, and offer to delete wrong-PM lockfiles.

---

## Step 4: Audit Build Scripts

Use the chosen package manager from Step 1 to determine the correct commands. Reference this table:

| PM | Restore deps | Add package | Runner |
|----|-------------|-------------|--------|
| npm | `npm ci --ignore-scripts` | `npm install --save-exact --ignore-scripts <pkg>` | `npx` |
| bun | `bun install --frozen-lockfile --ignore-scripts` | `bun add --exact --ignore-scripts <pkg>` | `bunx` |
| pnpm | `pnpm install --frozen-lockfile --ignore-scripts` | `pnpm add --save-exact --ignore-scripts <pkg>` | `pnpm dlx` |
| yarn | `yarn install --frozen-lockfile --ignore-scripts` | `yarn add --exact --ignore-scripts <pkg>` | `yarn dlx` |

**Note:** `--ignore-scripts` may break projects that rely on postinstall steps (e.g. husky, node-gyp, prisma, esbuild native binaries). The final verification step will catch this — flag it to the operator if the build fails.

Scan all build/deploy locations (already identified in Step 1a) and flag THREE types of issues:

### Wrong package manager
If the operator chose to standardize in Step 1c, flag any **package management command** using the wrong PM — install commands, lockfile references, runner commands. Example: `npm ci` in a CI config → should be `bun install --frozen-lockfile --ignore-scripts`.

**Runtime and container changes are a separate concern.** Dockerfile base images (`FROM node:20-alpine` → `FROM oven/bun:1-alpine`), runtime CMD entries, and similar changes are NOT package management hygiene — they are runtime migration. Present these as additional migration/policy changes in a separate section, not as required fixes.

### Unsafe install commands
Flag any install command that:
- Does a bare install (no lockfile-only mode) — re-resolves the entire dependency tree
- Is missing `--ignore-scripts` — allows postinstall scripts to run

### Runner commands (`bunx` / `npx`)

Runner commands need special care. If the target package is already declared in the project, the safest approach is usually to rely on that declared dependency rather than introducing a separate version pin in the runner command. If the target package is not declared locally, an unpinned runner command may fetch and execute code outside the lockfile.

**Decision tree for each runner command:**

1. **Is the tool already declared as a dependency/devDependency** in the relevant `package.json` and present in the lockfile?
   - Yes → treat it as locally pinned. **Do NOT add `@version`** to the runner command.
   - Also flag the command if it uses the wrong runner for the chosen PM.

2. **Is the tool NOT declared locally?**
   - Flag it for review.
   - Preferred fix: add the package as a devDependency with an exact version, then invoke it through the project's normal scripts/tooling.
   - Acceptable fallback: pin the exact package name and version in the runner command if adding a dependency is not appropriate.

3. **Does the binary name differ from the package name?**
   - Verify the actual package name before recommending a change.
   - Do not assume the binary name is the npm package name, especially for scoped packages.
   - Always use the real scoped package name (`bunx @scope/package@version`), never guess from the binary name alone.

4. **Wrong runner for the chosen PM?** Flag it (e.g. `npx` when the project uses bun → should be `bunx`).

Present findings and offer to fix.

### Project-wide PM hardening (recommended, not required)

These are explicit checks — verify each one and report pass/fail. They are categorized as **recommended** in the summary, not required hygiene fixes.

**Secret handling exception for project-root `.npmrc`:** it is allowed to probe the project-root `.npmrc` for the exact keys `ignore-scripts` and `min-release-age` using an anchored search that only returns matching lines. Do not print or inspect any other `.npmrc` content, and do not read user-level or global `.npmrc` files.

| # | Check | How to verify | If missing |
|---|-------|--------------|------------|
| 1 | Project-root `.npmrc` exists with `ignore-scripts=true` | Use an anchored search on the project-root `.npmrc` for `ignore-scripts=true`. For npm projects, `npm config get ignore-scripts --location=project` also works. Do NOT accept `bunfig.toml` `ignoreScripts` as a substitute — it is undocumented. The official approach for both npm and bun is `.npmrc`. | Create/update the project-root `.npmrc` with `ignore-scripts=true`. This enforces `--ignore-scripts` project-wide for all developers, not just Codex. Works for both npm and bun projects (bun reads `.npmrc`). |
| 2 | `minimumReleaseAge` is configured | For bun: check `bunfig.toml` for `minimumReleaseAge` under `[install]`. For npm: run `npm config get min-release-age --location=project`. | Recommend adding. Bun: `bunfig.toml` with `[install] minimumReleaseAge = 604800` (7 days in seconds). Exclusions via `minimumReleaseAgeExcludes`. npm: project-root `.npmrc` with `min-release-age=7` (7 days). Requires npm `11.10.0+` — if older, note the upgrade requirement. |

Present findings as a pass/fail table alongside the other Step 4 findings, but mark them as **(recommended)** so the operator knows they are hardening, not required fixes.

If `bunfig.toml` contains `ignoreScripts` but the project-root `.npmrc` does not contain `ignore-scripts=true`, Check 1 is still MISSING. Do not upgrade it to PASS based on `bunfig.toml` alone.

### Additional migration changes (if standardizing PM)

If the operator chose to standardize on a PM in Step 1c, these runtime changes may also be needed for the PM switch to work end-to-end in this repo. Present them in a separate section — they carry more breakage risk than PM hygiene fixes and are not required for supply-chain hygiene:

- **Runtime/container migration** — Dockerfile base images (`FROM node:20-alpine` → `FROM oven/bun:1-alpine`), CMD entries, and similar runtime changes.

Do not mark these as FAIL. Present them as additional migration changes and let the operator choose whether to apply them alongside the required fixes.

---

## Step 5: Scan for Secret Files

Scan the filesystem for files that likely contain secrets. **Do not read their contents — detection is by filename only.**

Use `find` or Glob to locate files matching these patterns in the project root (exclude `node_modules`):

```
.env, .env.*, .env.local, .env.production
.dev.vars
credentials*, *secret*, *token*
*.pem, *.key, *.p12
.vscode/launch.json
```

You MUST actually search the filesystem — do not just check `.gitignore` and assume. A file can exist on disk without being in `.gitignore`, or be in `.gitignore` but not covered by Codex project instructions.

For each file found on disk, check TWO things:
1. **Is it ignored by git?** — use `git check-ignore -v <path>`. If not ignored, flag as FAIL
2. **Is it covered by `AGENTS.md` secret-handling instructions?** — if not, flag as FAIL

Present findings as a table:

| File | Ignored by git | In AGENTS.md secret-handling instructions |
|------|----------------|-------------------------------------------|
| .env | ✓ | ✗ — needs adding |
| .dev.vars | ✓ | ✓ |

Offer to fix both git ignore rules (for example in `.gitignore`) and `AGENTS.md`.

---

## Step 6: Audit Project AGENTS.md

Check if the project has an `AGENTS.md` file at the project root. If it exists, scan for supply chain and safety reminders. If missing or incomplete, offer to add:

- Package manager preference (e.g. "This project uses bun. Never use npm." — based on the operator's choice in Step 1)
- Supply chain safety (ci/frozen-lockfile commands, --save-exact, --ignore-scripts, lockfile commitment)
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
- Package versions: X pinned, Y need fixing
- Lockfile: <status>
- Build scripts: X clean, Y need fixing
- PM hardening (recommended): .npmrc ignore-scripts <status>, minimumReleaseAge <status>
- Additional migration changes: X may be needed to complete PM standardization in this repo
- Secret files: X protected, Y need fixing
- Project AGENTS.md: <status>
```

Then ask whether the operator wants to apply:

- only the required hygiene fixes
- the required hygiene fixes plus any additional migration/policy changes
- one group at a time

After applying fixes, recommend the operator run the chosen PM's locked install command to verify the pinned versions resolve cleanly:

- npm: `npm ci`
- bun: `bun install --frozen-lockfile`
- pnpm: `pnpm install --frozen-lockfile`
- yarn: `yarn install --frozen-lockfile`

If the operator wants deeper assurance, suggest separate follow-up work such as vulnerability scanning — `npm audit` for npm projects, or a third-party tool like `snyk` or `osv-scanner` for bun/pnpm projects (bun has no built-in audit command). Make clear that this is different from pinning and lockfile hygiene.

---

## Important

- **Never read secret file contents.** Detection is by filename only.
- **Always report before fixing.** The operator decides what to change.
- **Be specific about what changes.** Show the exact edit before making it.
- **This is a point-in-time audit.** Ongoing enforcement comes from hooks (bash-guard.sh, tkt-remind.sh) and project instructions (AGENTS.md). This skill sets up the project; hooks maintain it.
