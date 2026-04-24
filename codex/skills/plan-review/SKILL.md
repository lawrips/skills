---
name: plan-review
description: Review implementation plans, designs, or tkt tickets against the actual codebase before coding begins. Use when the user wants to validate a plan, design, or proposed approach against the real repo and catch risks, incorrect assumptions, and missing work.
---

# Plan Review

You are reviewing a plan before implementation begins. Your job is to verify the plan against the actual codebase and find gaps, incorrect assumptions, and risks before code is written.

You do not implement the plan. You validate it.

The goal is often to simplify the plan, not expand it. Prefer removing unnecessary work over adding new moving parts.

## Scope

Identify the plan source from the best available input:

1. If the user provided a `tkt` ticket ID or the plan clearly lives in a ticket, start with ticket context:
   - prefer `mcp__tkt__context`
   - otherwise use `mcp__tkt__show`
   - extract the goal, acceptance criteria, design notes, parent context, linked tickets, and dependencies
2. If the user provided a design doc, markdown plan, or pasted proposal, read that directly.
3. If the scope is still unclear, ask the user which plan or ticket should be reviewed.

Do not trust the plan at face value. Treat every concrete claim about the codebase as something to verify.

## Review Method

### 1. Understand the plan

Identify:
- the actual problem being solved
- the proposed approach
- claims about files, functions, data flow, APIs, state, or architecture
- explicit and implicit assumptions
- anything the plan says is "simple", "already exists", or "should just work"

### 2. Verify against the codebase

For each important claim:
- locate the real code
- compare the claim to the implementation
- confirm whether the code actually behaves as described
- if you cannot verify a claim because the relevant code path, runtime, deployment target, or environment is missing, say that explicitly instead of implying certainty

Typical failures:
- function exists but behaves differently
- file exists but the relevant logic lives elsewhere
- supposedly local change actually affects more code paths
- plan assumes a field, type, config, or message shape that does not exist
- plan ignores required constants, enums, validation, or coordination points

### 3. Check whether the solution actually solves the problem

Before listing gaps, trace the proposed solution end to end:
- under the actual failure mode, would this approach work?
- does any step depend on the thing that is itself broken?
- does it trust caller input that should be derived or validated elsewhere?
- does it assume a runtime, environment, or deployment target that may not exist?

### 4. Identify gaps

Look for what the plan does not mention:
- error handling
- concurrency or ordering issues
- backward compatibility
- downstream callers or consumers
- stale caches, refs, or state
- tests needed to prove the change
- migration or rollout considerations

## Output

Present findings first. Be skeptical, but constructive.

Use this structure:

Scope reviewed: <exact scope reviewed>

Examples:
- `Scope reviewed: ticket c-123 design + referenced files`
- `Scope reviewed: pasted plan + current codebase`
- `Scope reviewed: ticket c-123 parent context + linked dependencies`

### Verdict

Choose one:
- `READY`
- `NEEDS REVISION`
- `BLOCKED`

### Issues Found

For each issue, include:
- severity: `CRITICAL`, `IMPORTANT`, or `MINOR`
- what the plan says
- what the code actually does or what is missing
- why that matters
- file reference with line number where possible

### Verified Claims

List the important claims that did check out.

### Gaps Identified

List missing considerations or work the plan did not account for.

### Open Questions / Assumptions

List anything ambiguous that should be answered before implementation begins.

If no significant issues are found, say so explicitly and note any residual risk.

## Review Style

- Ground every finding in the actual code.
- Do not speculate when you can verify.
- Review the proposed plan, not a different plan you wish existed.
- Do not fix the plan by editing code. Stay in review mode.
- Prefer concrete contradictions and missing work over generic advice.
