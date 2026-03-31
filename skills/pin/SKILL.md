---
name: pin
description: Audit and lock down a project's supply chain. Pins package versions, ensures lockfiles are committed, flags secret files, and hardens build scripts. Use when setting up a new project, onboarding a repo, or after a dependency scare.
disable-model-invocation: true
compatibility: Node.js / Bun projects. Claude Code only.
metadata:
  author: lawrips
  version: 1.3.0
---

# Pin

You are auditing a project's supply chain hygiene. Work through each check in order, report findings as you go, and offer to fix each issue. Do not read the contents of any file that may contain secrets.

## Principles

- **Scan, don't read secrets.** Use `ls`, `find`, and `git ls-files` to detect secret files. Never `cat` or `Read` their contents.
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
2. **Keep mixed** — the skill audits both but notes the risk (Claude may use the wrong one, lockfiles may drift out of sync).

Wait for the answer before proceeding. The chosen PM determines the correct commands for Steps 2–6.

If the operator chooses to standardize, flag the wrong PM's lockfile for removal and all wrong-PM commands for migration as findings in Steps 3 and 4.

---

## Step 2: Audit Package Versions

Scan `package.json` for unpinned versions in `dependencies` and `devDependencies`. Flag any version that starts with `^`, `~`, `>=`, `>`, `*`, or `latest`.

Present findings as a table:

| Package | Current | Issue |
|---------|---------|-------|
| axios | ^1.14.0 | Range — pins to 1.14.0 |
| lodash | ~4.17.0 | Range — pins to 4.17.0 |

To determine the correct pinned version, check the lockfile for the currently resolved version. If the lockfile doesn't exist yet (new project), fall back to `npm view <pkg> version` to get the current latest (this works regardless of which PM the project uses). Offer to fix all at once.

---

## Step 3: Audit Lockfile

Four checks for the chosen PM's lockfile:

1. **Lockfile exists** — does the lockfile for the chosen package manager exist? For bun, check both `bun.lock` and `bun.lockb` — if both exist, flag the old format (`bun.lockb`) for removal.
2. **Lockfile is tracked by git** — run `git ls-files <lockfile>` to verify it's committed, not just present on disk
3. **Lockfile not gitignored** — scan `.gitignore` for the lockfile name. If the lockfile IS listed in `.gitignore`, flag it as a FAIL — lockfiles must be committed
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

### Project-level config file
Check if the project has a PM config file that enforces `--ignore-scripts` by default for all developers, not just Claude:
- npm: `.npmrc` with `ignore-scripts=true`
- bun: `bunfig.toml` with `[install] ignore-scripts = true`
- pnpm: `.npmrc` with `ignore-scripts=true`

If the file doesn't exist or doesn't have the setting, offer to create/update it. This ensures `--ignore-scripts` is the default for anyone working on the project — not dependent on remembering to pass the flag.

Scan all build/deploy locations (already identified in Step 1a) and flag THREE types of issues:

### Wrong package manager
If the operator chose to standardize in Step 1c, flag any command using the wrong PM. Example: project standardized on bun but `Dockerfile.dev` uses `npm ci` → flag and recommend `bun install --frozen-lockfile --ignore-scripts`.

### Unsafe install commands
Flag any install command that:
- Does a bare install (no lockfile-only mode) — re-resolves the entire dependency tree
- Is missing `--ignore-scripts` — allows postinstall scripts to run

### Runner commands (`bunx` / `npx`)

**CRITICAL: `bunx foo` and `bunx foo@version` have DIFFERENT behavior.** Without a version, `bunx`/`npx` resolves the binary locally from `node_modules/.bin`. With `@version`, it downloads from the npm registry. This means adding a version pin can *introduce* risk rather than reduce it.

**Decision tree for each runner command:**

1. **Is the package already a local dependency** (in `package.json` + lockfile)? → **Do NOT add `@version`.** The version is already pinned in `package.json`. The runner resolves locally. Leave it alone.

2. **Is the package NOT a local dependency?** → Pin it. But first:
   - Find the real package name — the binary name may differ from the package name (e.g. binary `opennextjs-cloudflare` is provided by package `@opennextjs/cloudflare`). Check `node_modules/.bin/<binary>` or search `package.json`.
   - Always use the scoped package name (`bunx @scope/package@version`), never the unscoped binary name — unscoped names may be squatter packages on npm.
   - Get the version from the registry (`npm view <package> version`).
   - Verify the package@version exists before recommending it.

3. **Wrong runner for the chosen PM?** Flag it (e.g. `npx` when the project uses bun → should be `bunx`).

Present findings and offer to fix.

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

You MUST actually search the filesystem — do not just check `.gitignore` and assume. A file can exist on disk without being in `.gitignore`, or be in `.gitignore` but not in `permissions.deny`.

For each file found on disk, check TWO things:
1. **Is it in .gitignore?** — if not, flag as FAIL
2. **Is it in `.claude/settings.local.json` permissions.deny?** — if not, flag as FAIL (needs Read/Write/Edit deny triple)

Present findings as a table:

| File | In .gitignore | In permissions.deny |
|------|--------------|-------------------|
| .env | ✓ | ✗ — needs adding |
| .dev.vars | ✓ | ✓ |

Offer to fix both `.gitignore` and `.claude/settings.local.json`. When adding to `settings.local.json`, use the deny triple pattern:

```json
"Read(<path>)", "Write(<path>)", "Edit(<path>)"
```

---

## Step 6: Audit Project CLAUDE.md

Check if the project has a `CLAUDE.md` file at the project root. If it exists, scan for supply chain and safety reminders. If missing or incomplete, offer to add:

- Package manager preference (e.g. "This project uses bun. Never use npm." — based on the operator's choice in Step 1)
- Supply chain safety (ci/frozen-lockfile commands, --save-exact, --ignore-scripts, lockfile commitment)
- Git safety (no destructive commands, no force push, no chained push)
- Secret handling (don't read .env files, ask user for secret values)

Do not overwrite existing content — append a new section if the file already exists.

---

## Summary

After all checks, present a summary:

```
Pin audit complete:
- Package versions: X pinned, Y need fixing
- Lockfile: <status>
- Build scripts: X clean, Y need fixing
- Secret files: X protected, Y need fixing
- Project CLAUDE.md: <status>
```

Offer to fix all issues at once or one at a time. After applying fixes, recommend the operator run `<pm> install --frozen-lockfile` (using the chosen PM) to verify the pinned versions resolve cleanly.

Optionally, suggest a vulnerability audit as a separate follow-up — `npm audit` for npm projects, or a third-party tool like `snyk` or `osv-scanner` for bun/pnpm projects (bun has no built-in audit command). This checks if any pinned versions have known CVEs, which is a different concern from supply chain pinning.

---

## Important

- **Never read secret file contents.** Detection is by filename only.
- **Always report before fixing.** The operator decides what to change.
- **Be specific about what changes.** Show the exact edit before making it.
- **This is a point-in-time audit.** Ongoing enforcement comes from hooks (bash-guard.sh, tkt-remind.sh) and permissions (settings.json). This skill sets up the project; hooks maintain it.
