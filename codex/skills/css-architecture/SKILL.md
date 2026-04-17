---
name: css-architecture
description: Use when creating or refactoring CSS, tokens, theming, or component styling in an existing repo. Helps keep styling consistent with the current design system while avoiding hardcoded colors, override chains, and token drift.
---

# CSS Architecture

Use this skill when working on styling in an existing repo.

This is not a redesign skill. Start by understanding the styling system that already exists, then extend or repair it with the smallest coherent change.

## What This Skill Is For

Use it for:
- new component styling
- theming work
- CSS refactors toward tokens
- fixing styling inconsistencies
- extracting repeated styling into shared patterns

Do not use it to invent a new styling architecture unless the repo clearly needs one and the user wants that change.

## Inspect First

Before making styling changes:

1. Find the current styling approach:
   - CSS modules
   - global CSS
   - Tailwind or utility classes
   - styled-components / emotion
   - theme JSON or token files
2. Find the token or theme source of truth.
3. Find one or two nearby components that already solve a similar styling problem.
4. Check whether the repo already has:
   - semantic tokens
   - context-specific token families
   - component variants
   - established spacing, radius, and typography scales

Briefly summarize what system you found before making broad styling changes.

## Choose the Work Mode

Classify the task before changing code:

### 1. New Styling

Use when the component or surface is new.

Goal:
- fit the existing system
- reuse existing tokens, classes, and patterns
- add new tokens only when the current system cannot express the need cleanly

### 2. Tokenization / Refactor

Use when styles exist but rely on hardcoded values, duplicated rules, or fragile selectors.

Goal:
- replace raw values with semantic tokens
- reduce duplication
- move theme logic into tokens/config instead of selector overrides

### 3. Theme / Contrast Fix

Use when styling breaks in a different theme, background context, or emphasis level.

Goal:
- fix the token mapping, not just the one visible instance
- make sure background/text pairs remain readable
- prefer context-specific tokens over one-off overrides

### 4. Component Extraction

Use when styling or markup is duplicated across multiple places.

Goal:
- extract shared structure and shared styles
- keep variant differences explicit
- avoid copy-pasted CSS that drifts over time

## Hard Rules

- Never hardcode colors in CSS when the repo has tokens or theme values.
- Never assume text will remain readable on a background; use paired background/text tokens.
- Never rely on theme override chains when token-driven styling can express the same thing.
- Never use appearance-based class names like `blue-button` or `large-text`.
- Never introduce inline styles for normal styling work unless the repo already depends on that pattern and the user wants it.
- Always prefer the repo's current design system over inventing a parallel one.

## Implementation Rules

When editing styles:

- Prefer semantic tokens over raw values.
- Prefer purpose-based names over appearance-based names.
- Reuse existing token families before introducing new ones.
- If you add a background token, add the matching text token when needed.
- Cover relevant states for interactive elements:
  - hover
  - focus-visible
  - active
  - disabled
  - loading or selected states where relevant
- Keep styling logic close to the system boundary:
  - tokens/config own theme complexity
  - component CSS should stay simple

## Refactor Rules

When cleaning up existing styling:

- Remove hardcoded color usage first.
- Collapse repeated styles into shared classes, shared components, or shared tokens.
- Replace fragile selector chains with token-driven styling where possible.
- Preserve behavior and visual intent while reducing complexity.
- Do not rename or reorganize styling primitives casually if the repo already has stable conventions.

The goal is usually to simplify the styling model, not expand it.

## Output Expectations

When you use this skill, report back:

- what styling system you found
- what files or components were changed
- what token family or theme source of truth was reused
- whether any new semantic tokens were introduced
- whether interaction states were covered
- any remaining styling debt or follow-up cleanup worth doing

## References

For deeper patterns, use `references/advanced-patterns.md`:
- theme classification
- override elimination
- component extraction
- state coverage
- documentation and migration guidance
