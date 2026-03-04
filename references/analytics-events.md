---
title: Analytics & FPI Events
impact: HIGH
impactDescription: Duplicate or misplaced events break platform analytics, attribution, and merchant dashboards
tags: analytics, fpi-events, tracking, ecommerce, gtm
---

## Analytics & FPI Events

The FPI store emits standardized e-commerce analytics events consumed by Fynd's tracking infrastructure, Google Tag Manager, and merchant analytics dashboards. Incorrectly firing events breaks attribution, conversion tracking, and the merchant's reporting.

## Core Rules

### 1. Fire Events Only at the Correct User Interaction Point

Each event has exactly one intended trigger. Firing it elsewhere creates duplicate data.

| Event | Correct Trigger |
|-------|----------------|
| `product_viewed` | When product detail page renders and product data is first loaded |
| `add_to_cart` | Immediately after a successful `fpi.cart.addItems()` response |
| `remove_from_cart` | Immediately after successful cart item removal |
| `checkout_started` | When user lands on the checkout page |
| `purchase` | After successful order placement confirmation |
| `search` | After user submits a search query |
| `product_list_viewed` | When a PLP/collection page renders with its product list |

### 2. Never Fire Events in Effects That Can Re-run

`useEffect` without a stable dependency array runs on every render. Analytics effects must have precise dependencies.

**Incorrect (fires `product_viewed` on every re-render):**

```jsx
useEffect(() => {
  fpi.analytics.fireEvent("product_viewed", { product });
}); // no dependency array — runs every render
```

**Incorrect (fires again when unrelated state changes):**

```jsx
useEffect(() => {
  fpi.analytics.fireEvent("product_viewed", { product });
}, [product, cartCount]); // cartCount shouldn't trigger this
```

**Correct (fires once when product data first loads):**

```jsx
const hasTracked = useRef(false);

useEffect(() => {
  if (product && !hasTracked.current) {
    hasTracked.current = true;
    fpi.analytics.fireEvent("product_viewed", { product });
  }
}, [product]);
```

### 3. Never Fire Events During SSR

Analytics events require browser context (cookies, localStorage, GTM). Guard all event calls.

**Incorrect:**

```jsx
// Fires during SSR — no browser context, no GTM, corrupts session data
fpi.analytics.fireEvent("page_view", data);
```

**Correct:**

```jsx
useEffect(() => {
  // useEffect is browser-only — safe
  fpi.analytics.fireEvent("page_view", data);
}, []);
```

### 4. Don't Duplicate Events Already in Existing Hooks

The theme's existing hooks in `theme/helper/hooks/` and page components already fire the standard events. Before adding a new event call, read the existing hook/page implementation to confirm it isn't already tracked.

```jsx
// Before adding this, check if useProductDetails hook already fires it
fpi.analytics.fireEvent("product_viewed", ...);
```

### 5. Use FPI Event Methods — Not GTM Direct Calls

Don't push directly to `window.dataLayer` or call GTM functions. The FPI analytics layer normalizes event data and handles the GTM integration.

**Incorrect:**

```jsx
window.dataLayer.push({ event: "add_to_cart", ... });
```

**Correct:**

```jsx
fpi.analytics.fireEvent("add_to_cart", {
  product,
  quantity,
  price: product.price.effective.min,
});
```
