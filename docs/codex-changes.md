# Codex Changes

Short reference for the Codex-specific setup added so far.

## Global Instructions

File: `~/.codex/AGENTS.md`

Current additions:
- secret-handling rules
- `tkt` workflow reminder
- Bun-by-default package manager guidance
- JS / TS supply-chain defaults
- git safety rules

Purpose:
- set global behavior defaults for Codex across projects
- keep repo-specific instructions in local `AGENTS.md` files when needed

## Installed Skills

### `brainstorming`

File: `~/.codex/skills/brainstorming/SKILL.md`

Purpose:
- opt-in brainstorming only when explicitly requested
- grounded in `tkt` context, recent commits, and relevant code
- iterative discovery before recommendation
- section-by-section design with ticket creation at the end

Typical invocation:
- `$brainstorming <topic>`

### `code-review`

File: `~/.codex/skills/code-review/SKILL.md`

Purpose:
- review diffs, commits, PRs, or `tkt` ticket implementations
- findings first
- acceptance-criteria check when a ticket exists
- explicit `Scope reviewed` line
- explicit verification notes for tests / dynamic verification
- separate follow-up suggestions from actual findings

Typical invocation:
- `$code-review <ticket-or-scope>`

### `plan-review`

File: `~/.codex/skills/plan-review/SKILL.md`

Purpose:
- validate plans or ticket designs against the actual codebase before implementation
- ticket-aware via `tkt` context
- explicit `Scope reviewed` line
- simplification bias: reduce unnecessary work rather than expand the plan
- explicit rule to say when a claim could not be verified

Typical invocation:
- `$plan-review <ticket-or-plan>`

### `pin`

File: `~/.codex/skills/pin/SKILL.md`

Purpose:
- baseline supply-chain hygiene audit for Node/Bun and Python projects
- checks package-manager consistency, pinned direct dependencies, lockfiles, build scripts, hardening policy, secret-file handling, and project agent guidance
- uses ecosystem references for Node/Bun and Python details
- frames Socket/Bun scanner support as recommended hardening, not a comprehensive security guarantee

Typical invocation:
- `$pin`

### `css-architecture`

File: `~/.codex/skills/css-architecture/SKILL.md`

Purpose:
- preserve the existing styling system in a repo while improving consistency
- inspect token and theme sources first
- guide new styling, token refactors, theme fixes, and shared component extraction
- avoid hardcoded colors, token drift, and override-chain theming

Typical invocation:
- `$css-architecture <scope>`

## Rules

File: `~/.codex/rules/default.rules`

Current addition:
- `git push` now requires approval

Rule:

```python
prefix_rule(
    pattern=["git", "push"],
    decision="prompt",
    justification="Always require approval before pushing to a remote.",
)
```

Notes:
- a broader `git` allow rule still exists, but `git push` resolves to `prompt`
- `codex execpolicy check` confirmed the final decision is `prompt`

## Restart

After changing skills or rules, restart Codex so the current session picks them up reliably.
