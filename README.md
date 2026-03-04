# Skills

Personal Claude Code skills and agents collection - sharing what has worked for me. Use at own risk.

## Installation

```bash
/plugin marketplace add lawrips/skills
/plugin install skills@lawrips
```

## Prerequisites

Several skills and agents are tightly integrated with [tkt](https://github.com/lawrips/tkt), a ticket management CLI. They won't work as expected without it:
- **Skills:** `create-tickets`
- **Agents:** `plan-reviewer`, `code-reviewer`

## Skills

| Skill | Description |
|-------|-------------|
| **brainstorming** | Turn ideas into designs through step-by-step Q&A |
| **investigate** | Debugging methodology that prevents speculation and enforces disciplined investigation |
| **create-tickets** | Convert designs into tkt epics and tasks |
| **css-architecture** | CSS token system and semantic styling patterns |
| **docker-dev-setup** | Isolated Docker dev environment with security hardening |

## Agents

| Agent | Description |
|-------|-------------|
| **surgical-coder** | Implements features and fixes with surgical precision, matching existing patterns |
| **plan-reviewer** | Validates implementation plans against the actual codebase before coding begins |
| **code-reviewer** | Reviews code changes against ticket requirements and codebase conventions |

## Workflow

Skills and agents split into two categories: **process** (the steps you follow) and **deliberate** (tools you reach for when needed).

### Process — feature lifecycle

For large items like epics I do something similar to the following flow.

```
          ┌──────────────────┐
  skill   │  brainstorming   │
          └────────┬─────────┘
                   │ design
                   ▼
          ┌──────────────────┐
  skill   │  create-tickets  │
          └────────┬─────────┘
                   │ tickets
                   ▼
          ┌──────────────────┐
  agent   │  plan-reviewer   │
          └────────┬─────────┘
                   │ ticket notes
                   ▼
          ┌──────────────────┐
  agent   │  surgical-coder  │
          └────────┬─────────┘
                   │ code
                   ▼
          ┌──────────────────┐
  agent   │  code-reviewer   │
          └────────┬─────────┘
                   │ review notes
                   ▼
          ┌──────────────────┐
    tkt   │ commit & close   │
          └──────────────────┘
```

### Deliberate — reach for when needed

| Skill/Agent | When |
|-------------|------|
| **investigate** | Debugging a bug — load it to reset Claude's approach |
| **css-architecture** | Major styling work — load conventions for the session |
| **docker-dev-setup** | Setting up containerized dev environment from scratch |

## License

MIT

## Inspiration 

Thank you to obra, pprice, smacbeth, wedow

