---
name: css-architecture
description: CSS token system and semantic styling patterns. Use when creating styles, theming, or fixing styling inconsistencies.
metadata:
  author: lawrips
  version: 1.2.0
---

# CSS Architecture & Design Systems Skill

This document captures detailed, reusable principles for building maintainable CSS architectures and design systems. These are generalizable patterns learned from real-world refactoring work.

---

## ⛔ HARD RULES — DO NOT BREAK

These rules are non-negotiable. Breaking them creates technical debt that compounds over time.

### 1. NEVER hardcode colors in CSS
```css
/* ❌ FORBIDDEN */
.button { background: #4A7C59; color: #fff; }

/* ✅ REQUIRED */
.button { background: var(--semantic-btn-bg); color: var(--semantic-btn-text); }
```

### 2. NEVER assume text will be readable on a background
```css
/* ❌ FORBIDDEN - assumes background will be dark */
.badge { background: var(--color-1); color: white; }

/* ✅ REQUIRED - text color from theme */
.badge { background: var(--semantic-badge-bg); color: var(--semantic-badge-text); }
```

### 3. NEVER use override chains for themes
```css
/* ❌ FORBIDDEN - this pattern explodes combinatorially */
.button { ... }
.theme-light .button { ... }
.theme-dark .button { ... }
.header .button { ... }
.theme-light .header .button { ... }

/* ✅ REQUIRED - one rule, theme handles complexity */
.button { background: var(--semantic-btn-bg); color: var(--semantic-btn-text); }
```

### 4. NEVER use inline styles
```jsx
/* ❌ FORBIDDEN */
<div style={{ backgroundColor: '#4A7C59', color: 'white' }}>

/* ✅ REQUIRED */
<div className="card-primary">
```

### 5. ALWAYS pair background and text tokens
If you define `--semantic-X-bg`, you MUST define `--semantic-X-text`. They are inseparable.

### 6. ALWAYS use context-appropriate tokens
```css
/* ❌ WRONG - content token in header */
.header-button { background: var(--semantic-content-btn-bg); }

/* ✅ CORRECT - header token in header */
.header-button { background: var(--semantic-header-btn-bg); }
```

### 7. NEVER use appearance-based class names
```css
/* ❌ FORBIDDEN */
.blue-button { ... }
.large-text { ... }

/* ✅ REQUIRED */
.btn-primary { ... }
.heading { ... }
```

---

## 1. Semantic Token Architecture

### The Problem
Hardcoding colors creates maintenance nightmares. Consider:

```css
/* Looks fine initially */
.button {
  background: #4A7C59;
  color: #fff;
}
```

This breaks when:
- You introduce a theme with a light primary color (white-on-white)
- You want to support dark mode
- You want users to customize colors

### The Solution: Purpose-Based Tokens

Instead of colors describing appearance, tokens describe **purpose**:

```css
/* Token describes what it's FOR, not what color it IS */
.button {
  background: var(--semantic-content-btn-bg);
  color: var(--semantic-content-btn-text);
}
```

### Token Naming Convention

```
--semantic-[context]-[element]-[property]
```

| Segment | Examples | Purpose |
|---------|----------|---------|
| `context` | `header`, `content` | Where in the UI |
| `element` | `btn`, `link`, `badge`, `text` | What type of element |
| `property` | `bg`, `text`, `border`, `hover-bg` | What CSS property |

**Good names:**
- `--semantic-header-btn-bg` → "background for buttons in the header"
- `--semantic-content-btn-accent-text` → "text color for accent buttons in content area"
- `--semantic-selected-bg` → "background for selected items"

**Bad names:**
- `--blue-button` → describes appearance
- `--color-3` → meaningless without context
- `--primary` → too vague, doesn't indicate usage context

### Layered Token System

Build tokens in layers:

```
Layer 1: Raw values (defined once, rarely used directly)
├── --color-1: #4A7C59
├── --color-2: #B5D99C
└── --color-3: #E8C07D

Layer 2: Derived values (computed from Layer 1)
├── --color-1-dark: [darken(color-1, 15%)]
├── --color-1-light: [lighten(color-1, 15%)]
└── --color-section-bg: [rgba(color-2, 0.12)]

Layer 3: Semantic tokens (what developers actually use)
├── --semantic-header-btn-bg: rgba(255, 255, 255, 0.1)
├── --semantic-content-btn-bg: var(--color-1)
└── --semantic-selected-bg: var(--color-1)
```

**Key insight**: Developers should almost never use Layer 1 or 2 directly. They use Layer 3 tokens which are guaranteed to work correctly for the current theme.

---

## 2. Context-Aware Styling

### The Problem
UI elements appear in different contexts with different background colors. A button that works on a dark header becomes invisible on a light content area.

### The Solution: Context-Specific Token Families

Define parallel token families for each context:

```css
/* Header context (colored background) */
--semantic-header-btn-bg: rgba(255, 255, 255, 0.1);
--semantic-header-btn-text: rgba(255, 255, 255, 0.9);
--semantic-header-btn-hover-bg: rgba(255, 255, 255, 0.2);

/* Content context (light background) */
--semantic-content-btn-bg: var(--color-1);
--semantic-content-btn-text: #ffffff;
--semantic-content-btn-hover-bg: var(--color-1-dark);
```

Each context gets a complete set of tokens. When a developer needs a button, they pick based on WHERE it appears:

```css
/* Button in header */
.header-button {
  background: var(--semantic-header-btn-bg);
  color: var(--semantic-header-btn-text);
}

/* Button in content area */
.content-button {
  background: var(--semantic-content-btn-bg);
  color: var(--semantic-content-btn-text);
}
```

### Accent/Emphasis Variants

Within each context, provide accent variants for attention-grabbing elements:

```css
/* Regular header button */
--semantic-header-btn-bg
--semantic-header-btn-text

/* Accent header button (needs attention, e.g., "Save" with unsaved changes) */
--semantic-header-btn-accent-bg
--semantic-header-btn-accent-text
```

This creates a decision tree for developers:
1. What context? (header vs content)
2. What emphasis? (regular vs accent)

---

## 3. Theme Type Classification

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

## 4. Eliminating Override Chains

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

## 5. Text Color Contrast Rules

### The Problem
Text on colored backgrounds needs appropriate contrast. `color: #fff` works on dark backgrounds but fails on light ones.

### The Solution: Paired Background/Text Tokens

ALWAYS define text color alongside background color:

```css
/* ❌ Wrong - assumes background will be dark */
.badge {
  background: var(--color-1);
  color: #fff;
}

/* ✅ Correct - text color comes from theme */
.badge {
  background: var(--semantic-badge-bg);
  color: var(--semantic-badge-text);
}
```

### Contrast Decision Rules

When defining theme tokens, use this heuristic:

| Background Luminance | Text Color |
|---------------------|------------|
| > 0.5 (light) | Dark text (`#2d3748`) |
| < 0.5 (dark) | White text (`#FFFFFF`) |

Common light colors needing dark text:
- Yellows/golds: `#E8C07D`, `#F4A460`, `#E6C17A`
- Light peaches: `#E8B4A0`, `#D4A574`
- Light greens: very muted/pastel greens

Common dark colors working with white text:
- Forest greens: `#4A7C59`, `#6B8F71`
- Blues: `#2D5A7B`, `#5E7B99`
- Reds/rusts: `#C75B39`, `#A45A3D`
- Grays: `#4A5568`, `#5A6B7A`

---

## 6. Component Extraction Patterns

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

## 7. Avoiding Common Anti-Patterns

### Anti-Pattern: Hardcoded Colors

```css
/* ❌ Will break on theme changes */
.special-button {
  background: #4A7C59;
  color: white;
}
```

**Fix**: Use semantic tokens

### Anti-Pattern: Context-Unaware Components

```css
/* ❌ Assumes it's always on light background */
.link {
  color: var(--color-1);
}
```

**Fix**: Create context-specific variants or use semantic tokens

### Anti-Pattern: Override Chains

```css
/* ❌ Fragile, hard to maintain */
.button { ... }
.theme-light .button { ... }
.theme-dark .button { ... }
.header .button { ... }
.theme-light .header .button { ... }
```

**Fix**: Move logic to theme tokens, keep CSS simple

### Anti-Pattern: Magic Numbers

```css
/* ❌ What does 0.7 mean? Why 0.7? */
.muted-text {
  opacity: 0.7;
}
```

**Fix**: Use design tokens
```css
.muted-text {
  color: var(--semantic-header-text-muted);
}
```

### Anti-Pattern: Inline Styles

```jsx
/* ❌ Breaks theming, can't be overridden, hard to maintain */
<div style={{ backgroundColor: '#4A7C59', color: 'white' }}>
```

**Fix**: Always use classes with CSS

### Anti-Pattern: Appearance-Based Class Names

```css
/* ❌ Describes what it looks like, not what it's for */
.blue-button { ... }
.large-text { ... }
.rounded-box { ... }
```

**Fix**: Use purpose-based names
```css
.btn-primary { ... }
.heading { ... }
.card { ... }
```

---

## 8. State Management in CSS

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

## 9. Documentation Requirements

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

## 10. Migration Strategy

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

---

## Summary Checklist

When building or reviewing CSS architecture:

- [ ] All colors use semantic tokens, not hardcoded values
- [ ] Tokens describe purpose, not appearance
- [ ] Each UI context has its own token family
- [ ] Text colors are paired with backgrounds
- [ ] No override chains based on theme classes
- [ ] Theme definitions contain all context-specific values
- [ ] Interactive elements have complete state coverage
- [ ] Shared UI is extracted into components
- [ ] Class names describe purpose, not appearance
- [ ] No inline styles
- [ ] Documentation exists for token usage
