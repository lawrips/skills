---
name: create-tickets
description: Convert designs into tkt epics and tasks for implementation. Use when user wants to break down a design into tickets, create an epic, or plan implementation tasks.
disable-model-invocation: true
compatibility: Requires tkt CLI (https://github.com/lawrips/tkt). Claude Code only.
metadata:
  author: lawrips
  version: 1.5.0
---

# Creating Tickets

Convert designs into actionable tickets using tkt's native fields.
Note: **Use friendly file names for tickets.** When creating tickets, use `--id` to set a descriptive kebab-case name: `tkt create "Title" --id c-network-retry-duplicates`. Much easier to scan and find than auto-generated IDs like `c-4c1b`.


## Prerequisites

Verify `tkt` is installed. If unavailable, stop and tell the user to install it.

## Assess Scope

- Single task → standalone ticket
- Multiple steps → epic with child tasks

## Creating an Epic

```bash
tkt create "Feature Name" \
  --id "Friendly name e.g. c-network-retry" \
  --type epic \
  -d "High-level goal" \
  --design "Architecture decisions, approach, constraints" \
  --acceptance "Definition of done"
```

### Creating Tasks

```bash
tkt create "Task title" \
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
tkt ls --parent <epic-id>  # show created tasks
tkt ready                   # confirm unblocked work
```

## Finalize

Tickets should be closed after implementation and only when verified and confirmed by the user. Use `tkt workflow` for this flow.

## Troubleshooting

- **Duplicate ID** — If `tkt create` fails with a duplicate, check `tkt show <id>` to see if the ticket already exists. Append a suffix or choose a different name.
- **Parent not found** — Verify the parent exists with `tkt show <parent-id>` before creating children.
- **MCP tools unavailable** — Fall back to bash `tkt` commands directly.

## Standalone Task

For small features without an epic:

```bash
tkt create "Task title" \
  --id "Friendly name e.g. c-network-retry" \
  -d "What to accomplish" \
  --design "Files and approach" \
  --acceptance "How to verify"
```
