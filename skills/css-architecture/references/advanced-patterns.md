# Advanced CSS Architecture Patterns

Deep-dive patterns for theming, refactoring, and component extraction. Consult these when working on specific scenarios — they build on the core principles in SKILL.md.

---

## Theme Type Classification

### The Problem
Different theme types require fundamentally different color strategies. A "dark mode" theme needs different defaults than a "light primary color" theme.

### The Solution: Explicit Theme Types

Classify each theme:

```json
{
  "id": "forest-meadow",
  "type": "normal",  // light | normal | dark
  ...
}
```

| Type | Primary Color | Header Text | Content Background |
|------|---------------|-------------|-------------------|
| `light` | White/very light | Dark | White |
| `normal` | Medium/dark colored | White | White |
| `dark` | Dark | Light | Dark |

Each type follows different token patterns:

**Light themes**: Use tertiary color (`color-3`) for interactive elements since primary is too light
**Normal themes**: Use primary color (`color-1`) for main actions, tertiary for accents
**Dark themes**: Invert expectations - light colors on dark backgrounds

---

## Eliminating Override Chains

### The Problem
CSS overrides create fragile, unmaintainable code:

```css
/* Base button */
.button {
  background: var(--color-1);
  color: #fff;
}

/* Override for light themes */
.theme-light .button {
  background: var(--color-3);
  color: #2d3748;
}

/* Override for dark mode */
.theme-dark .button {
  background: var(--color-3);
  color: #1a1a2e;
}

/* Override for this specific button on light theme header */
.theme-light .header .button {
  background: rgba(0, 0, 0, 0.1);
}
```

This explodes combinatorially. Every new component needs overrides for every theme.

### The Solution: Explicit Token Values Per Theme

Move ALL the logic into the theme definition:

```json
{
  "id": "light-theme",
  "semantic": {
    "headerBtnBg": "rgba(0, 0, 0, 0.05)",
    "headerBtnText": "#2d3748",
    "contentBtnBg": "#5A6B7A",
    "contentBtnText": "#FFFFFF"
  }
}
```

```json
{
  "id": "dark-theme",
  "semantic": {
    "headerBtnBg": "rgba(255, 255, 255, 0.1)",
    "headerBtnText": "#e8e8ec",
    "contentBtnBg": "#E8C39E",
    "contentBtnText": "#1A1A2E"
  }
}
```

Now CSS is simple and override-free:

```css
.header-button {
  background: var(--semantic-header-btn-bg);
  color: var(--semantic-header-btn-text);
}
```

**The theme handles the complexity, not the CSS.**

---

## Component Extraction Patterns

### The Problem
UI code gets duplicated across views, leading to inconsistency and maintenance burden.

### Recognition Signals

Watch for these patterns that indicate extraction is needed:

1. **Similar JSX structures** in multiple files
2. **Copy-pasted CSS** with minor variations
3. **Same props/data** rendered differently in different contexts
4. **Conditional rendering** that could be a prop

### Extraction Strategy

**Step 1**: Identify the common structure
```jsx
// Found in 3 different files:
<div className="item">
  <span className="bullet">•</span>
  <span className="text">{text}</span>
</div>
```

**Step 2**: Identify the variations
- Sometimes bullet is shown, sometimes not
- Class names differ (`note-item` vs `activity-item`)
- Sometimes text has link parsing

**Step 3**: Create component with props for variations
```jsx
function NoteItem({ text, showBullet = true, className = '', compact = false }) {
  const baseClass = compact ? 'activity-item' : 'note-item';
  return (
    <div className={`${baseClass} ${className}`}>
      {showBullet && <span className="bullet">•</span>}
      <span className="text">{parseLinks(text)}</span>
    </div>
  );
}
```

**Step 4**: Replace all instances with the component

### CSS Consolidation

When extracting, consolidate the CSS too:

```css
/* Before: duplicated with slight variations */
.note-item { display: flex; gap: 0.5rem; }
.activity-item { display: flex; gap: 0.375rem; }

/* After: shared base with variants */
.item-shared {
  display: flex;
  gap: var(--space-2);
}
.item-shared.compact {
  gap: var(--space-1);
}
```

---

## State Management in CSS

### Complete State Coverage

Every interactive element needs these states defined:

```css
.interactive-element {
  /* Default state */
  background: var(--semantic-btn-bg);
  color: var(--semantic-btn-text);

  /* Hover state */
  &:hover:not(:disabled) {
    background: var(--semantic-btn-hover-bg);
  }

  /* Focus state (keyboard navigation) */
  &:focus-visible {
    outline: 2px solid var(--semantic-focus-ring);
    outline-offset: 2px;
  }

  /* Active/pressed state */
  &:active:not(:disabled) {
    transform: scale(0.98);
  }

  /* Disabled state */
  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  /* Loading state */
  &.loading {
    pointer-events: none;
    opacity: 0.8;
  }
}
```

### Variant States

For elements with variants (selected, has-changes, error), use modifier classes:

```css
.tab {
  background: var(--semantic-header-btn-bg);
}

.tab.selected {
  background: var(--semantic-selected-bg);
  color: var(--semantic-selected-text);
}

.button.has-changes {
  background: var(--semantic-header-btn-accent-bg);
  color: var(--semantic-header-btn-accent-text);
}

.input.error {
  border-color: var(--semantic-error-border);
}
```

---

## Documentation Requirements

### Theme Documentation

Every theming system needs:

1. **Token reference table** - All tokens with purpose and example usage
2. **Decision tree** - How to choose the right token
3. **Theme type guide** - How light/normal/dark themes differ
4. **Code examples** - Copy-paste patterns for common cases
5. **Anti-patterns** - What NOT to do
6. **Testing checklist** - What to verify when creating themes

### Component Documentation

For shared components:

1. **Props table** - All props with types and defaults
2. **Usage examples** - Common use cases
3. **Styling hooks** - How to customize appearance
4. **Accessibility notes** - ARIA requirements

---

## Migration Strategy

When refactoring existing CSS to a token-based system:

### Phase 1: Audit
1. Find all hardcoded colors (`grep -r "#[0-9a-fA-F]" --include="*.css"`)
2. Find all `color:` rules
3. Identify patterns (which colors are used where)

### Phase 2: Define Tokens
1. Create semantic token names based on usage patterns
2. Define token values for each theme type
3. Add tokens to theme configuration

### Phase 3: Apply Tokens
1. Add :root defaults in CSS
2. Apply tokens via theme context/provider
3. Replace hardcoded values with tokens
4. Delete any override chains that become unnecessary

### Phase 4: Clean Up
1. Remove unused CSS rules
2. Remove legacy theme detection logic
3. Update documentation

**Key principle**: Do a clean cutover, not incremental migration. Partial migrations create two systems to maintain.
