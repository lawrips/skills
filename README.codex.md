# Codex

This repo is primarily Claude-oriented, but it also includes Codex-ready versions of a few workflows.

## Included Codex Skills

- `brainstorming`
- `css-architecture`
- `code-review`
- `plan-review`

These live in `codex/skills/`.

## Install

Follow the instructions in [`codex/INSTALL.md`](/Users/lawrence/git/basato-skills/codex/INSTALL.md).

The short version:
- clone this repo somewhere on disk
- symlink or copy the skill folders from `codex/skills/` into `~/.codex/skills/`
- restart Codex

## Optional Global Setup

If you want the broader Codex behavior changes as well, see [`docs/codex-changes.md`](/Users/lawrence/git/basato-skills/docs/codex-changes.md).

That covers:
- suggested global `~/.codex/AGENTS.md` defaults
- the `git push` approval rule in `~/.codex/rules/default.rules`
- the currently installed local skills
