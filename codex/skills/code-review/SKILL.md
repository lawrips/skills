---
name: code-review
description: Review code changes, diffs, commits, pull requests, or tkt ticket implementations for bugs, regressions, missing tests, blast radius, and scope creep. Use when the user asks for a review, asks "wdyt of this diff", or wants a ticket implementation reviewed without immediately fixing it.
---

# Code Review

You are reviewing implementation changes. Your job is to find problems, not to fix them.

Default to a code review mindset:
- prioritize correctness, regressions, missing tests, blast radius, and scope creep
- use the ticket or stated requirements as the review contract whenever they exist
- ground every finding in the actual diff and surrounding code
- do not modify files unless the user explicitly asks for fixes after the review

## Scope

Identify the review scope from the best available source, in this order:

1. If the user provided a `tkt` ticket ID or the review is clearly about a ticket, start with ticket context:
   - prefer `mcp__tkt__context`
   - otherwise use `mcp__tkt__show`
   - extract the goal, acceptance criteria, design notes, linked tickets, and any linked commits
2. If the user provided specific commits, a PR, a diff, or files, inspect that exact scope.
3. If the user asked for a review without specifying scope, inspect the current changes:
   - `git status --short`
   - `git diff --cached`
   - `git diff`
4. If the scope is still unclear, ask the user what should be reviewed.

Prefer exact diffs over summaries. Read surrounding code only as needed to understand behavior, contracts, and blast radius.

## Ticket-First Review

When a `tkt` ticket exists, do not review the diff in isolation.

Use the ticket to define expected behavior:
- read the ticket first
- read any parent or design context needed to understand scope
- if commits are linked in the ticket context, inspect those commits directly
- if no commits are linked, infer scope from the current diff and any files referenced in the ticket
- explicitly check each acceptance criterion against the implementation
- flag scope creep separately from defects

If the ticket and the code disagree, call that out directly.

## Review Method

For each changed area, check:

- **Correctness**: does the change actually do what it appears to intend?
- **Regressions**: what existing behavior could break?
- **Requirements fit**: if the user gave requirements or a ticket, were they actually met?
- **Blast radius**: what callers, consumers, or adjacent code paths are affected?
- **Tests**: are tests missing, stale, or no longer proving the right behavior?
- **Consistency**: does the change match the repo's existing patterns and abstractions?
- **Scope creep**: were unrelated changes mixed in?

Common review traps to watch for:

- layered fixes that patch symptoms instead of root cause
- duplicated logic instead of reusing existing paths
- silent contract changes in data shape, function signature, or config
- string literals or hardcoded values where shared constants should be used
- rewrites that drop behavior the old code handled
- changes that look local but require coordinated updates elsewhere

## Output

Present findings first, ordered by severity.

Use this structure:

Scope reviewed: <exact scope reviewed>

Examples:
- `Scope reviewed: staged diff`
- `Scope reviewed: commit abc1234`
- `Scope reviewed: ticket c-123 linked commits + current diff`
- `Scope reviewed: working tree diff in pages/index.tsx and lib/auth.ts`

### Findings

Each finding should include:
- severity: `CRITICAL`, `IMPORTANT`, or `MINOR`
- what is wrong
- why it matters
- file reference with line number

Only include actual defects, regressions, requirement misses, or concrete risks in Findings. Do not put weak suggestions or nice-to-have cleanup here.

### Acceptance Criteria Check

Include this section when a ticket or explicit requirements exist.

For each acceptance criterion, state:
- `met`
- `partially met`
- `not met`

Keep each explanation brief and concrete.

### Verification Notes

Always state:
- whether tests were run, and which ones
- whether any relevant code paths were verified dynamically

If tests were not run, say so explicitly.
If relevant behavior was not verified dynamically, say so explicitly.
This section is required so confidence stays calibrated.

### Open Questions / Assumptions

List anything that could change the review outcome or needs confirmation.

### Follow-up Suggestions

Use this section for non-blocking suggestions, cleanup ideas, or out-of-scope improvements.
Keep these separate from Findings.

### Change Summary

Keep this brief and only include it after findings.

If no findings are discovered, say that explicitly and mention any residual risks or testing gaps.

## Review Style

- Be direct and specific. Do not pad with praise.
- Cite files and lines for every material claim.
- Review what changed, not what you wish had been built.
- Prefer a small number of strong findings over a long list of weak nits.
- If something is only a suggestion and not really a defect, put it in Follow-up Suggestions, not Findings.
