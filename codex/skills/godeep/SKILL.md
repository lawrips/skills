---
name: godeep
description: Plan long-running unattended system-health work when the user invokes /godeep or asks for overnight, deep work, maintenance, optimization, documentation, performance, cleanup, or high-token planning. Use to inspect tickets, epics, code, docs, and recent context; propose concrete unattended work candidates; let the user refine or accept one; create a tkt epic with executable child tickets; run plan-review automatically; and incorporate review findings into the epic for later /execute.
---

# Go Deep

`/godeep` is a planning skill for neglected, high-leverage system-health work. It does not implement the work. It discovers, proposes, refines, and packages an executable tkt epic that can later be handed to `/execute`.

The target use case is an unattended long run: overnight, a large token budget, or a "developer has a quiet week with no meetings and no new feature releases" window.

## Refresh Context

Start by gathering current project context.

Use tkt first:
- run `tkt workflow` before ticket-management work, preferring MCP when available
- inspect old, open, blocked, recently closed, and recently modified tickets
- inspect existing epics, dependency chains, and recurring themes
- read any relevant ticket context that suggests neglected maintenance or systemic issues

Then inspect the repo enough to calibrate proposals:
- recent commits and active branch context
- docs, setup guides, architecture notes, and workflow docs
- tests, CI config, lint/typecheck setup, and validation scripts
- TODO/FIXME notes when discoverable
- code areas with duplication, broad queries, repeated work, flaky tests, stale docs, or unclear boundaries

Do not make code changes in this skill.

## Candidate Selection

Look for work that is:
- long-running but bounded
- decomposable into clear tickets
- useful if partially completed
- low on product-decision risk
- reviewable from commits and tests
- likely to improve system health, reliability, performance, documentation, or developer experience
- suitable for unattended `/execute` work after planning

Avoid work that needs ongoing user taste, prioritization, or live decisions.

### Good Candidate Examples

These examples calibrate the kind of work that fits. They are not a whitelist.

- Verified docs refresh: review architecture, setup, ticket, or workflow docs against the actual codebase and update stale parts.
- Test health: add missing regression coverage, improve brittle tests, fix flaky tests, or make validation easier to run.
- Performance improvements with clear targets: reduce duplicate DB calls, narrow overly broad queries, avoid repeated work, improve slow user-input responsiveness, or remove obvious render/update bottlenecks.
- Internal refactors with stable behavior: consolidate duplicated logic, simplify confusing modules, or improve boundaries without changing product behavior.
- Developer experience maintenance: improve scripts, local setup, CI reliability, lint/typecheck consistency, or troubleshooting docs.
- Cleanup after verification: remove dead code, obsolete flags, unused paths, stale tickets, abandoned docs, or TODOs where intended behavior is clear.

### Bad Candidate Examples

These examples calibrate what to avoid unless the user explicitly narrows the work into an executable maintenance package.

- New product features that need user taste, prioritization, or interactive decisions.
- Visual redesigns or UX changes without a clear design spec.
- Broad rewrites where correctness depends on subjective review.
- Auth, billing, permissions, deletion, or migration work without explicit acceptance criteria.
- Major dependency or framework upgrades with unclear compatibility fallout.
- Vague goals like "make it faster", "clean up the codebase", or "improve architecture" without a measurable target or bounded area.

## Propose Options

Propose a small set of concrete candidates, usually 3-6. Make each option specific enough that the user can accept, reject, combine, or refine it.

For each candidate include:
- title
- why it is a good `/godeep` candidate
- expected payoff
- likely scope and affected areas
- unattended fit
- decision risk
- suggested ticket breakdown
- validation strategy
- risks or reasons not to do it now

Prefer a handful of sharp proposals over a broad list of chores.

## Refine With The User

This skill may be interactive. Let the user choose a candidate, combine candidates, narrow scope, or ask for more options.

If the user already provides a clear target, skip broad proposal generation and turn that target into an executable epic plan.

Do not create the epic until the user accepts a direction or the user's request clearly authorizes creating one.

## Create The Epic

Once the user accepts a direction, create a tkt epic and child tickets according to the current `tkt workflow`.

Write the epic for unattended execution:
- state the goal and non-goals
- include design/context needed by a future `/execute` run
- define acceptance criteria
- create child tickets with concrete scopes
- add validation notes to each ticket
- add dependencies or ordering edges
- avoid ambiguous tickets that require live product decisions
- include assumptions and safe boundaries

The output should be an executable work package, not just a planning note.

## Run Plan Review

After creating the epic and child tickets, automatically run `$plan-review` against the created epic.

Use the review findings to revise the epic package:
- update epic description, design, and acceptance criteria
- update child ticket scopes and validation notes
- add missing tickets when the review finds real gaps
- add or adjust dependencies
- remove or narrow risky work that does not fit unattended execution
- preserve review concerns that cannot be fully resolved as explicit assumptions or risks

If the review triggers substantial changes, run a second `$plan-review` pass when useful. Do not implement code.

## Final Output

Finish by reporting:
- created epic ID/title
- child tickets created
- dependency/order summary
- plan-review result and incorporated changes
- assumptions and residual risks
- suggested `/execute <epic>` command
