# Advanced CSS Architecture Patterns

Deep-dive patterns for theming, refactoring, and component extraction. Consult these when working on specific scenarios — they build on the core principles in `SKILL.md`.

## Theme Type Classification

Different theme types require different defaults. A light-primary theme, a normal theme, and a dark theme do not want the same token mappings.

Classify themes explicitly where the repo supports it:

```json
{
  "id": "forest-meadow",
  "type": "normal"
}
```

Typical patterns:

| Type | Primary Color | Header Text | Content Background |
|------|---------------|-------------|-------------------|
| `light` | very light | dark | light |
| `normal` | medium/dark | light | light |
| `dark` | dark | light | dark |

Use the theme type to drive token choices, not selector overrides.

## Eliminate Override Chains

Avoid patterns like:

```css
.button { ... }
.theme-light .button { ... }
.theme-dark .button { ... }
.header .button { ... }
.theme-light .header .button { ... }
```

This becomes unmaintainable as themes and contexts multiply.

Prefer moving those decisions into token values or theme config:

```css
.header-button {
  background: var(--semantic-header-btn-bg);
  color: var(--semantic-header-btn-text);
}
```

The theme owns the value selection. The component CSS stays simple.

## Component Extraction Signals

Extraction is usually warranted when you see:

- similar JSX or template structure in multiple files
- copy-pasted CSS with only minor differences
- repeated rendering of the same data shape
- conditional branches that should probably be props

Extraction approach:

1. identify the shared structure
2. identify the real variants
3. extract a shared component
4. consolidate the CSS alongside it

Do not extract too early if the cases are only superficially similar.

## State Coverage

Interactive elements should usually define:

- default
- hover
- focus-visible
- active
- disabled
- loading / selected / error when relevant

Example:

```css
.interactive-element {
  background: var(--semantic-btn-bg);
  color: var(--semantic-btn-text);
}

.interactive-element:hover:not(:disabled) {
  background: var(--semantic-btn-hover-bg);
}

.interactive-element:focus-visible {
  outline: 2px solid var(--semantic-focus-ring);
  outline-offset: 2px;
}
```

Do not ship interactive UI that only looks correct in its default state.

## Token Design Guidance

Prefer semantic token naming:

```text
--semantic-[context]-[element]-[property]
```

Examples:

- `--semantic-header-btn-bg`
- `--semantic-content-btn-text`
- `--semantic-selected-bg`

Avoid:

- `--blue-button`
- `--color-3` as an app-facing token
- vague names like `--primary` without context

If a repo already has a different stable naming convention, follow that system instead of forcing this one.

## Background / Text Pairing

Background and text contrast are inseparable.

If you add or change a background token that carries text, make sure the matching text token is defined and used. Avoid assumptions like `color: white` unless that is genuinely guaranteed by the design system.

## Documentation and Migration

When the styling system changes materially, document:

- what the token source of truth is
- how to choose the right token family
- what anti-patterns were removed
- what migration work remains

When refactoring:

- prefer staged cleanup over sweeping renames
- preserve behavior while reducing styling complexity
- make the next change easier instead of merely moving code around
