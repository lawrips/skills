---
name: execute
description: Complete a referenced tkt epic end to end when the user invokes /execute or asks to execute an epic. Use for autonomous ticket execution workflows where Codex should resolve the epic, prepare a clean working tree, follow the current tkt workflow, implement each ticket in dependency order, validate, run a final code-review automatically, incorporate concrete fixes, commit, and finish with a summary instead of stopping for clarification.
---

# Execute

`/execute` is a completion skill for tkt epics. Treat invocation as authorization to work through the referenced epic end to end.

Prefer progress with documented assumptions over pausing for clarification. Do not ask clarifying questions during execution; make reasonable, educated assumptions, continue through issues when possible, and call out assumptions, tradeoffs, blocked items, skipped validation, and unresolved risks in the final summary.

## Starting State

Begin by inspecting the working tree.

If there is uncommitted work, preserve it before starting the epic:
- review the current changes well enough to group them logically
- commit existing work in one or more clear logical commits
- do not discard, overwrite, or revert user work
- avoid mixing pre-existing work with the epic's ticket commits
- if some local change cannot safely be committed, isolate it as much as possible, continue with independent work when feasible, and report it in the final summary

Start epic execution from the cleanest achievable working tree.

## Workflow Refresh

Run `tkt workflow` before any ticket-management work. Prefer the tkt MCP workflow tool when available; otherwise use the CLI.

Do not copy or restate ticket lifecycle rules in this skill. Read the workflow output and follow it as authoritative for ticket status updates, notes, dependencies, and closure behavior.

## Epic Resolution

Resolve the `/execute <epic>` argument as the referenced tkt epic. Use the best available tkt source:
- prefer MCP tools for workflow, epic, context, dependency, and ticket operations
- use the tkt CLI when MCP does not provide the needed operation
- if the argument is not an exact ID, search by friendly name/title and choose the strongest match

Open the epic and gather enough context to execute it:
- epic description, design, and acceptance criteria
- child tickets and their statuses
- dependency tree and ready order
- linked tickets or parent context that materially affect execution
- recent commits when relevant

## Ticket Order

Work one ticket at a time in dependency order. Prefer tickets whose dependencies are complete. If multiple tickets are ready, choose the order that best preserves system coherence and keeps commits reviewable.

Do not skip a ticket merely because details are imperfect. Make a reasonable implementation assumption, document it, and keep moving unless the ticket is genuinely impossible to progress.

When a ticket is blocked but independent tickets remain, continue with the independent ready work and report the blocked ticket at the end.

## Per-Ticket Execution

For each ticket:

1. Read the ticket context and acceptance criteria.
2. Inspect the relevant code, tests, docs, and existing patterns.
3. Implement the ticket to completion within its intended scope.
4. Run focused validation for the changed area.
5. Run broader validation when the change touches shared behavior, public contracts, or cross-module flows.
6. Update tkt according to the workflow output.
7. Commit the completed ticket before starting the next one.

Keep commits aligned to ticket boundaries unless two tickets are inseparable. Use clear commit messages that make the ticket relationship obvious.

Do not push.

## Review And Fix Loop

Run `$code-review` once at the end of the epic by default, after all independently completable tickets have been implemented, validated, and committed.

Use `$code-review` as an automatic review pass, not as a stopping point. Treat `/execute` as authorization to fix concrete issues found by review, rerun relevant validation, and commit those fixes before finishing.

Prioritize:
- correctness bugs
- acceptance-criteria misses
- regressions
- missing tests for changed behavior
- scope creep that should be removed or isolated

Do not churn on weak suggestions. If a review comment is subjective, low-value, or conflicts with the ticket, make a reasonable call, document it in the summary, and keep moving.

Use targeted earlier `$code-review` only for unusually high-risk tickets, such as shared contract changes, tricky state/concurrency logic, validation failures, or large cross-module refactors. Otherwise preserve momentum and rely on the final epic review.

Commit final review fixes separately when they do not belong cleanly to a single ticket.

## Completion Criteria

Continue until the epic is complete or no further independent progress is possible.

Before finishing:
- confirm the working tree state
- confirm all completed ticket work has been committed
- run the final `$code-review`
- confirm code-review findings were either fixed and committed or explicitly documented
- update the epic according to the workflow output when its tickets are complete
- run any final validation that is appropriate for the full epic

## Final Summary

Finish with a concise execution summary and then wait for user review/testing.

Include:
- epic resolved
- tickets completed, skipped, or blocked
- commits made
- code-review passes and fixes incorporated
- tests/checks run and their results
- assumptions made
- issues worked around
- validation skipped or still needed
- residual risks or user review notes
