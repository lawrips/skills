---
name: create-tickets
description: Convert ideas into tk epics and /or tasks for implementation
---

# Creating Tickets

Convert designs into actionable tickets using tk's native fields.
Note: **Use friendly file names for tickets.** When creating tickets, use `--id` to set a descriptive kebab-case name: `tk create "Title" --id c-network-retry-duplicates`. Much easier to scan and find than auto-generated IDs like `c-4c1b`.


## Prerequisites

Verify `tk` is installed. If unavailable, stop and tell the user to install it.

## Assess Scope

- Single task → standalone ticket
- Multiple steps → epic with child tasks

## Creating an Epic

```bash
tk create "Feature Name" \
  --id "Friendly name e.g. c-network-retry" \
  --type epic \
  -d "High-level goal" \
  --design "Architecture decisions, approach, constraints" \
  --acceptance "Definition of done"
```

### Creating Tasks

```bash
tk create "Task title" \
  --parent <epic-id> \
  --id "Friendly name e.g. c-check-websocket-connection" \
  -d "What to accomplish" \
  --design "Files: src/foo.ts, src/bar.ts
Approach: specific implementation details" \
  --acceptance "Tests pass, integrates with X"
```

## Task Guidelines

- Each task: 2-5 minutes of work
- Include exact file paths in design
- Specific implementation steps, not vague instructions
- Clear acceptance criteria
- Do NOT ask for confirmation at any point. Create all tickets, set all dependencies, and run all commands without pausing.

## After Creating

```bash
tk ls --parent <epic-id>  # show created tasks
tk ready                   # confirm unblocked work
```

## Finalize

Tickets should be closed after implementation and only when verified and confirmed by the user. Commit ids shoudl be written to the ticket info for easier tracking

/commit-and-close details this flow

## Standalone Task

For small features without an epic:

```bash
tk create "Task title" \
  --id "Friendly name e.g. c-network-retry" \
  -d "What to accomplish" \
  --design "Files and approach" \
  --acceptance "How to verify"
```