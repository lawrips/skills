# Node / Bun Reference

Use this reference when Step 1 detects Node, Bun, npm, pnpm, or yarn signals.

## Package Manager Signals

**Lockfiles:**
- `package-lock.json` → npm
- `bun.lockb` OR `bun.lock` → bun
- `pnpm-lock.yaml` → pnpm
- `yarn.lock` → yarn

**Commands to scan in build/deploy files:**
- `npm`, `npx`
- `bun`, `bunx`
- `pnpm`, `pnpm dlx`
- `yarn`, `yarn dlx`

Runner commands (`npx`, `bunx`, `pnpm dlx`, `yarn dlx`) are package-manager signals.

## Version Audit

Scan every `package.json` in the project, including nested packages and workspaces.

Check:
- `dependencies`
- `devDependencies`

Flag any direct dependency version that starts with:
- `^`
- `~`
- `>=`
- `>`
- `*`
- `latest`

To determine the correct pinned version, check the lockfile for the currently resolved version. If the lockfile does not exist yet, fall back to `npm view <pkg> version` to get the current latest. This works regardless of which Node package manager the project uses.

## Lockfile Audit

Expected lockfile by chosen PM:

| PM | Lockfile |
|----|----------|
| npm | `package-lock.json` |
| bun | `bun.lock` or `bun.lockb` |
| pnpm | `pnpm-lock.yaml` |
| yarn | `yarn.lock` |

For bun, check both `bun.lock` and `bun.lockb`. If both exist, flag the old format (`bun.lockb`) for removal.

If the operator chose to standardize on one package manager, flag lockfiles from the other Node package managers for removal.

## Correct Commands

| PM | Restore deps | Add package | Runner |
|----|-------------|-------------|--------|
| npm | `npm ci --ignore-scripts` | `npm install --save-exact --ignore-scripts <pkg>` | `npx` |
| bun | `bun install --frozen-lockfile --ignore-scripts` | `bun add --exact --ignore-scripts <pkg>` | `bunx` |
| pnpm | `pnpm install --frozen-lockfile --ignore-scripts` | `pnpm add --save-exact --ignore-scripts <pkg>` | `pnpm dlx` |
| yarn | `yarn install --frozen-lockfile --ignore-scripts` | `yarn add --exact --ignore-scripts <pkg>` | `yarn dlx` |

`--ignore-scripts` may break projects that rely on postinstall steps such as husky, node-gyp, prisma, or esbuild native binaries. The final verification step will catch this — flag it to the operator if the build fails.

## Build Script Findings

### Wrong package manager

If the operator chose to standardize, flag any package management command using the wrong PM:
- install commands
- lockfile references
- runner commands

Runtime/container changes are separate. Dockerfile base images (`FROM node:20-alpine` → `FROM oven/bun:1-alpine`), runtime CMD entries, and similar changes are not package management hygiene. Present them as additional migration/policy changes, not required fixes.

### Unsafe install commands

Flag any install command that:
- Does a bare install with no lockfile-only mode
- Is missing `--ignore-scripts`

Examples:
- `npm install` in CI/deploy → should usually be `npm ci --ignore-scripts`
- `bun install` in CI/deploy → should usually be `bun install --frozen-lockfile --ignore-scripts`
- `pnpm install` in CI/deploy → should usually be `pnpm install --frozen-lockfile --ignore-scripts`

### Runner commands

Decision tree for each runner command:

1. **Is the tool already declared as a dependency/devDependency** in the relevant `package.json` and present in the lockfile?
   - Yes → treat it as locally pinned. Do **not** add `@version` to the runner command.
   - Still flag the command if it uses the wrong runner for the chosen PM.

2. **Is the tool not declared locally?**
   - Flag it for review.
   - Preferred fix: add the package as a devDependency with an exact version, then invoke it through the project's normal scripts/tooling.
   - Acceptable fallback: pin the exact package name and version in the runner command if adding a dependency is not appropriate.

3. **Does the binary name differ from the package name?**
   - Verify the actual package name before recommending a change.
   - Do not assume the binary name is the npm package name, especially for scoped packages.
   - Always use the real scoped package name (`bunx @scope/package@version`), never guess from the binary name alone.

4. **Wrong runner for the chosen PM?**
   - Flag it, e.g. `npx` when the project uses bun should be `bunx`.

## Project-Wide PM Hardening

These checks are recommended, not required.

**Secret handling exception for project-root `.npmrc`:** it is allowed to probe the project-root `.npmrc` for the exact keys `ignore-scripts` and `min-release-age` using an anchored search that only returns matching lines. Do not print or inspect any other `.npmrc` content, and do not read user-level or global `.npmrc` files.

| # | Check | How to verify | If missing |
|---|-------|--------------|------------|
| 1 | Project-root `.npmrc` exists with `ignore-scripts=true` | Use an anchored search on the project-root `.npmrc` for `ignore-scripts=true`. For npm projects, `npm config get ignore-scripts --location=project` also works. Do NOT accept `bunfig.toml` `ignoreScripts` as a substitute — it is undocumented. The official approach for both npm and bun is `.npmrc`. | Create/update the project-root `.npmrc` with `ignore-scripts=true`. This enforces `--ignore-scripts` project-wide for all developers, not just the agent. Works for both npm and bun projects (bun reads `.npmrc`). |
| 2 | `minimumReleaseAge` is configured | For bun: check `bunfig.toml` for `minimumReleaseAge` under `[install]`. For npm: run `npm config get min-release-age --location=project`. | Recommend adding. Bun: `bunfig.toml` with `[install] minimumReleaseAge = 604800` (7 days in seconds). Exclusions via `minimumReleaseAgeExcludes`. npm: project-root `.npmrc` with `min-release-age=7` (7 days). Requires npm `11.10.0+` — if older, note the upgrade requirement. |

If `bunfig.toml` contains `ignoreScripts` but the project-root `.npmrc` does not contain `ignore-scripts=true`, Check 1 is still MISSING. Do not upgrade it to PASS based on `bunfig.toml` alone.

## Verification Commands

After fixes, recommend the operator run:

- npm: `npm ci`
- bun: `bun install --frozen-lockfile`
- pnpm: `pnpm install --frozen-lockfile`
- yarn: `yarn install --frozen-lockfile`

If the operator wants deeper assurance, suggest separate follow-up work such as vulnerability scanning:
- `npm audit` for npm projects
- third-party tools like `snyk` or `osv-scanner` for bun/pnpm projects

Make clear that this is different from pinning and lockfile hygiene. Bun has no built-in audit command.
