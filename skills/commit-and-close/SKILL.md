---
name: commit-and-close
description: Commit code, update ticket with commit id, close ticket
---

# Commit and Close

After implementation is verified by the user, commit the code and close the ticket.

## Prerequisites

- User has confirmed the implementation works
- Changes are ready to commit

## Steps

1. **Commit the code** following the project's branch strategy (e.g. direct to main, feature branch, PR — match what the project typically does)

```bash
git add <files>
git commit -m "description of change"
```

2. **Add commit reference to the ticket**

```bash
tk add-note <ticket-id> "Implemented in commit $(git rev-parse --short HEAD)"
```

3. **Close the ticket**

```bash
tk edit <ticket-id> -s closed
```

4. **Confirm** — show the user the closed ticket

```bash
tk show <ticket-id>
```

## Rules

- Do NOT close tickets unless the user has verified the implementation
- Do NOT skip adding the commit reference — it's the traceability link
- If there's a parent epic, check if all child tasks are now closed. If so, ask the user if the epic should be closed too
