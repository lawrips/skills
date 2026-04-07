---
name: css-architecture
description: CSS token system and semantic styling patterns. Use when creating styles, theming, or fixing styling inconsistencies.
metadata:
  author: lawrips
  version: 1.4.0
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

## 3. Text Color Contrast Rules

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

## 4. Avoiding Common Anti-Patterns

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

## Advanced Patterns

For deeper patterns (theme classification, override elimination, component extraction, state management, migration strategy, documentation requirements), see `references/advanced-patterns.md`.

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
