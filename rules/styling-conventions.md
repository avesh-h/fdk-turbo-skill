---
title: Styling Conventions — Less Modules & CSS Variables
impact: MEDIUM
impactDescription: Inconsistent styling breaks the theme customization system and causes CSS leakage
tags: less, css-modules, css-variables, theming, styling
---

## Styling Conventions — Less Modules & CSS Variables

This codebase uses **Less modules** exclusively for component styling. Do not introduce `styled-components`, `emotion`, Tailwind, or inline style objects for layout/design.

## Less Modules — Scoped Component Styles

Every component has a co-located `.module.less` file. Class names are locally scoped by the Webpack build.

```
theme/components/product-card/
├── product-card.jsx
└── product-card.module.less
```

```less
// product-card.module.less
.card {
  border-radius: 8px;
  overflow: hidden;

  &:hover .image {
    transform: scale(1.05);
  }
}

.image {
  transition: transform 0.3s ease;
}

.price {
  color: var(--text-primary);
  font-family: var(--font-body);
}
```

```jsx
import styles from "./product-card.module.less";

function ProductCard() {
  return (
    <div className={styles.card}>
      <img className={styles.image} ... />
      <span className={styles.price} ... />
    </div>
  );
}
```

## CSS Variables for Theme-Configurable Values

Merchant-configurable values (colors, fonts, spacing) must be CSS variables — set once by `ThemeProvider`, consumed everywhere in Less.

**Incorrect (hardcoded color — merchant can't change it):**

```less
.button {
  background-color: #ff5733; // hardcoded
}
```

**Correct (CSS variable — merchant controls via theme settings):**

```less
.button {
  background-color: var(--button-primary-bg);
  color: var(--button-primary-text);
  font-family: var(--font-body);
}
```

Common CSS variables set by ThemeProvider:
- `--font-header`, `--font-body` — typography
- `--text-primary`, `--text-secondary`, `--text-disabled` — text colors
- `--button-primary-bg`, `--button-primary-text` — CTA buttons
- `--header-bg`, `--footer-bg` — layout colors
- `--accent-color` — brand accent

## Global Styles — Use Sparingly

Global styles go only in `theme/styles/base.global.less`. Reserve this for:
- CSS resets and base element styles (`body`, `*`, `a`, `h1`–`h6`)
- CSS variable declarations (`:root { --font-body: ... }`)
- Third-party library overrides

Never add component-specific styles to global Less files.

## Responsive — Mobile First

```less
.card {
  // Mobile base styles
  width: 100%;
  padding: 12px;

  // Tablet+
  @media (min-width: 768px) {
    width: 50%;
    padding: 16px;
  }

  // Desktop+
  @media (min-width: 1024px) {
    width: 33.33%;
  }
}
```

## Inline Styles — Only for Truly Dynamic Values

Inline `style` objects are acceptable only for values computed at runtime from data.

```jsx
// Acceptable — dynamic position from JS calculation
<div style={{ left: `${cursorX}px`, top: `${cursorY}px` }} />

// Acceptable — dynamic height from JS measurement
<div style={{ height: contentHeight }} />

// NOT acceptable — use Less module instead
<div style={{ padding: "16px", borderRadius: "8px" }} />
```

## Combining Class Names Conditionally

Use string concatenation or a utility — this codebase does not use `clsx` by default, check if it's imported in the component before adding it.

```jsx
// Without clsx
<div className={`${styles.card} ${isActive ? styles.active : ""}`} />

// With clsx (only if already used in file)
import clsx from "clsx";
<div className={clsx(styles.card, { [styles.active]: isActive })} />
```
