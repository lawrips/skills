# Python Reference

Use this reference when Step 1 detects Python packaging signals.

## Package Manager Signals

**Lockfiles and generated lock outputs:**
- `uv.lock` → uv
- `poetry.lock` → Poetry
- `pdm.lock` → PDM
- `Pipfile.lock` → Pipenv
- `requirements*.txt` / `constraints*.txt` with exact pins → pip or pip-tools

**Manifest and input files:**
- `pyproject.toml`
- `setup.cfg`
- `setup.py`
- `requirements*.in`
- `Pipfile`

**Commands to scan in build/deploy files:**
- `uv`, `uvx`
- `python -m pip`, `pip`, `pip3`, `pipx`
- `pip-compile`, `pip-sync`
- `poetry`
- `pdm`
- `pipenv`
- `hatch`
- `tox`
- `nox`

## Standardization

If multiple Python package managers are detected, ask the operator whether to standardize or keep mixed.

If the operator asks for a recommendation and there is no established Python package manager, recommend `uv` because it supports `pyproject.toml`, `uv.lock`, exact syncs, and direct dependency management in one workflow.

If the project already has an established Python package manager and lockfile, respect it unless the operator chooses to migrate.

## Version Audit

Scan direct dependencies in the files that apply to the detected package manager.

For `pyproject.toml`, check:
- `[build-system].requires`
- `[project].dependencies`
- `[project.optional-dependencies]`
- `[dependency-groups]`
- `[tool.poetry.dependencies]`
- `[tool.poetry.group.*.dependencies]`
- `[tool.pdm.dev-dependencies]`

For requirements-style projects, check:
- `requirements*.in`
- direct, hand-authored `requirements*.txt` files
- `constraints*.txt`

For Pipenv, check:
- `Pipfile` `[packages]`
- `Pipfile` `[dev-packages]`

For `setup.cfg`, check dependency fields such as `install_requires` and extras.

For `setup.py`, scan text only. Do not execute it.

Flag any direct dependency that is not exact-pinned. Examples:
- `requests>=2`
- `requests~=2.32`
- `requests>2`
- `requests<3`
- `requests`
- `requests = "*"` in Pipfile
- Poetry caret/tilde constraints such as `^2.0` or `~2.0`
- Build requirements such as `hatchling` without `==`

Treat these as exact pins:
- `package==1.2.3`
- Pipenv exact version values such as `"==1.2.3"`
- Lockfile entries for transitive dependencies

Direct URL, path, editable, and VCS dependencies need manual review. Flag them unless they are pinned to an immutable artifact, commit, or verified local path that the operator explicitly accepts.

To determine the correct pinned version, check the chosen PM's lockfile first:
- uv: `uv.lock`
- Poetry: `poetry.lock`
- PDM: `pdm.lock`
- Pipenv: `Pipfile.lock`
- pip-tools: generated `requirements*.txt` / `constraints*.txt`

If there is no lockfile yet, use the package manager's dry-run or index lookup to find the currently selected version. Examples:
- uv: `uv pip install --dry-run <pkg>`
- pip: `python -m pip index versions <pkg>`
- Poetry/PDM/Pipenv: use the package manager's add/lock dry-run if available, or resolve once and report that this will create a new lockfile

## Lockfile Audit

Expected lockfile by chosen PM:

| PM | Lockfile |
|----|----------|
| uv | `uv.lock` |
| Poetry | `poetry.lock` |
| PDM | `pdm.lock` |
| Pipenv | `Pipfile.lock` |
| pip-tools | generated `requirements*.txt` or `constraints*.txt` |

For pip-tools, the generated requirements or constraints file is the lock artifact. Prefer hash-enabled outputs when practical.

If the operator chose to standardize on one Python package manager, flag lockfiles from the other Python package managers for removal.

If the project has only `pyproject.toml`, `setup.py`, `setup.cfg`, `requirements.in`, or `Pipfile` with no lockfile or generated pinned requirements output, flag the missing lockfile as FAIL.

## Correct Commands

| PM | Restore deps | Add package | Runner |
|----|-------------|-------------|--------|
| uv | `uv sync --locked` | `uv add --exact <pkg>` | `uv run` |
| pip-tools | `pip-sync requirements.txt` | Add exact direct dependency to input, then `pip-compile --generate-hashes` | project venv command |
| Poetry | `poetry sync` | `poetry add <pkg>==<version>` | `poetry run` |
| PDM | `pdm sync --clean` | `pdm add <pkg>==<version>` | `pdm run` |
| Pipenv | `pipenv sync` | `pipenv install <pkg>==<version>` | `pipenv run` |

Adjust restore commands for project extras/dev groups:
- uv projects may need `uv sync --locked --all-extras` or explicit `--extra <name>`.
- Pipenv projects may need `pipenv sync --dev`.
- Poetry and PDM projects may need group flags depending on the repo.

Python package managers do not have a universal `--ignore-scripts` equivalent. Do not flag missing `--ignore-scripts` in Python commands.

## Build Script Findings

### Wrong package manager

If the operator chose to standardize, flag any package management command using the wrong Python PM:
- install commands
- lockfile references
- runner commands

Example: a uv project using `pip install -r requirements.txt` in CI should usually use `uv sync --locked` instead.

### Unsafe install commands

Flag commands that can re-resolve, ignore the lockfile, or install undeclared code:
- `pip install <pkg>` in build/CI/deploy scripts
- `pip install -r requirements.in`
- `pip install -r requirements.txt` when the file is not exact-pinned
- `uv sync` in CI/deploy without `--locked` or `--frozen`
- `uv run` in CI/deploy without `--locked`, if the command may update the lockfile
- `poetry install` when no `poetry.lock` exists
- `pdm install` when no `pdm.lock` exists
- `pipenv install` when `pipenv sync` or `pipenv install --deploy --ignore-pipfile` is intended
- `hatch run`, `tox`, or `nox` commands that create environments from unpinned dependencies
- `uvx`, `pipx run`, or similar runners for undeclared tools

Prefer locked restore commands from the table above.

### Runner commands

Decision tree for each runner command:

1. **Is the tool declared as a dependency/dev dependency and present in the lockfile?**
   - Yes → treat it as locally pinned.
   - Prefer the project runner: `uv run`, `poetry run`, `pdm run`, `pipenv run`, or the local virtualenv executable.

2. **Is the tool not declared locally?**
   - Flag it for review.
   - Preferred fix: add it as an exact-pinned dev dependency, regenerate the lockfile, and invoke it through the project runner.
   - Acceptable fallback: pin the runner invocation if the package manager supports exact pins.

3. **Does the binary name differ from the package name?**
   - Verify the actual package name before recommending a change.
   - Do not assume the CLI binary name is the package name.

Examples:
- `uvx ruff` in CI → prefer exact-pinned `ruff` dev dependency plus `uv run ruff`
- `pipx run black` in CI → prefer exact-pinned `black` dev dependency plus the project runner
- `python -m pytest` is acceptable when `pytest` is declared and locked in the project environment

## Project-Wide PM Hardening

These checks are recommended, not required.

| # | Check | How to verify | If missing |
|---|-------|--------------|------------|
| 1 | Locked install documented for the chosen PM | Check project instructions and CI/build scripts for the restore command from this reference. | Recommend adding the locked restore command to project instructions and CI. |
| 2 | Release-age policy documented or enforced, where supported | For uv, look for documented use of `uv lock --exclude-newer <date>` or an equivalent project policy. For other Python PMs, note whether there is an equivalent local policy. | Recommend documenting a release-age policy if the team wants one. For uv, use `--exclude-newer <date>` when intentionally refreshing locks. |
| 3 | Hash-checking used for pip-tools/pip outputs, where practical | For pip-tools, check whether generated requirements include hashes from `pip-compile --generate-hashes`. | Recommend `pip-compile --generate-hashes` for pip-tools projects when compatible with the project's package sources. |

Do not mark a project as FAIL because Python lacks Node-style `--ignore-scripts`.

## Verification Commands

After fixes, recommend the operator run the chosen PM's locked install command:

- uv: `uv sync --locked`
- pip-tools: `pip-sync requirements.txt`
- Poetry: `poetry sync`
- PDM: `pdm sync --clean`
- Pipenv: `pipenv sync`

Adjust for dev groups and extras as needed.

If the operator wants deeper assurance, suggest separate follow-up work such as vulnerability scanning or package trust review:
- `pip-audit`
- `osv-scanner`
- `snyk`
- Socket package/repo scanning, if the project uses Socket

Make clear that this is different from pinning and lockfile hygiene.
