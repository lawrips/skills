# Install Codex Skills

This repo includes Codex-ready versions of:

- `brainstorming`
- `css-architecture`
- `code-review`
- `plan-review`

## Recommended Install

Symlink the skills into `~/.codex/skills` so updates in this repo are reflected automatically.

Set your local repo path first:

```bash
REPO=/absolute/path/to/this/repo
```

Then install:

```bash
mkdir -p ~/.codex/skills
ln -s "$REPO/codex/skills/brainstorming" ~/.codex/skills/brainstorming
ln -s "$REPO/codex/skills/css-architecture" ~/.codex/skills/css-architecture
ln -s "$REPO/codex/skills/code-review" ~/.codex/skills/code-review
ln -s "$REPO/codex/skills/plan-review" ~/.codex/skills/plan-review
```

Restart Codex after installing.

## Copy Instead of Symlink

If you do not want symlinks:

```bash
REPO=/absolute/path/to/this/repo
mkdir -p ~/.codex/skills
cp -R "$REPO/codex/skills/brainstorming" ~/.codex/skills/
cp -R "$REPO/codex/skills/css-architecture" ~/.codex/skills/
cp -R "$REPO/codex/skills/code-review" ~/.codex/skills/
cp -R "$REPO/codex/skills/plan-review" ~/.codex/skills/
```

Restart Codex after copying.

## Notes

- These are local skills, not a Claude-style plugin marketplace install.
- `~/.codex/skills` is the reliable install target used in this setup.
- If a skill already exists at that destination, remove or rename it first.
- Optional global behavior and rules are documented in `docs/codex-changes.md`.
