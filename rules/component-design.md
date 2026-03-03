---
title: Component Design Patterns
impact: MEDIUM
impactDescription: Poor component design creates prop-drilling, duplicated logic, and hard-to-maintain UI
tags: components, hooks, reusability, dom-sanitization, sections
---

## Component Design Patterns

## Check Before Creating

Before writing a new component, hook, or utility, check what already exists:

| Need | Check First |
|------|------------|
| Custom hooks | `theme/helper/hooks.jsx` — `useThemeConfig`, `useAddress`, `usePolling`, `useIntersectionObserver`, etc. |
| Utilities | `theme/helper/utils.js` — image sizing, sanitization, color conversion, formatting |
| UI primitives | `theme/components/` — carousel, modal, loader, form fields, stars/rating, breadcrumb, etc. |
| Data resolvers | `theme/helper/lib.js` — `globalDataResolver`, `pageDataResolver` |

## Component Size

Keep components under ~300 lines. Split into sub-components when they grow. A good split point: when you can name a meaningful sub-component.

```jsx
// Too large — one component doing too much
function ProductDetailPage() {
  // 400+ lines: images, info, variants, add-to-cart, reviews, related...
}

// Better — split by responsibility
function ProductDetailPage() {
  return (
    <>
      <ProductGallery media={media} />
      <ProductInfo product={product} />
      <VariantSelector variants={variants} onSelect={handleVariant} />
      <AddToCartBar product={product} selectedVariant={variant} />
      <ProductReviews productId={product.uid} />
    </>
  );
}
```

## Avoid Prop Drilling Beyond 2 Levels

If a value needs to pass through more than 2 component levels, use `useGlobalStore` or create a context.

```jsx
// BAD — drilling theme config 3+ levels deep
<Page themeConfig={themeConfig}>
  <Section themeConfig={themeConfig}>
    <Card themeConfig={themeConfig}>
      <Price themeConfig={themeConfig} />
    </Card>
  </Section>
</Page>

// GOOD — Price reads config directly from the store
function Price({ amount }) {
  const { global_config } = useThemeConfig(); // reads from context
  const currency = global_config?.custom?.props?.currency?.value;
  return <span>{currency}{amount}</span>;
}
```

## Sanitize Platform HTML Content

CMS content, product descriptions, and platform HTML must be sanitized before rendering. Never use `dangerouslySetInnerHTML` on unsanitized strings.

**Incorrect:**

```jsx
<div dangerouslySetInnerHTML={{ __html: product.description }} />
```

**Correct (using DOMPurify — already in utils.js):**

```jsx
import { sanitizeHTML } from "../../helper/utils";

<div dangerouslySetInnerHTML={{ __html: sanitizeHTML(product.description) }} />
```

**Better (using html-react-parser for React component rendering):**

```jsx
import parse from "html-react-parser";
import { sanitizeHTML } from "../../helper/utils";

<div>{parse(sanitizeHTML(product.description))}</div>
```

## Sections vs Components

**Sections** (`theme/sections/`):
- Connected to FPI store — fetch their own data
- Declare a `props` schema for merchant configuration
- Are the units merchants drag-and-drop in the theme builder
- Should work standalone without parent passing data down

**Components** (`theme/components/`):
- Pure / presentational — receive data via props
- Reused across multiple sections and pages
- No direct FPI store connections (unless it's a smart component wrapping a dumb one)

## Functional Components Only

No class components. All state via hooks.

## Hoist Static JSX

JSX that doesn't depend on props or state should be defined outside the component to avoid recreation on every render.

**Incorrect:**

```jsx
function IconButton({ onClick }) {
  const icon = <svg>...</svg>; // recreated every render
  return <button onClick={onClick}>{icon}</button>;
}
```

**Correct:**

```jsx
const ICON = <svg>...</svg>; // created once at module level

function IconButton({ onClick }) {
  return <button onClick={onClick}>{ICON}</button>;
}
```

Reference: `vercel-react-best-practices/rules/rendering-hoist-jsx.md`
