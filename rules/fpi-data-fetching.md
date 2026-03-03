---
title: FPI Data Fetching Patterns
impact: CRITICAL
impactDescription: Incorrect patterns cause waterfalls, stale data, or missed SSR renders
tags: fpi, graphql, fdk-store, data-fetching, ssr, swr
---

## FPI Data Fetching Patterns

All platform data (products, cart, user, orders, config, catalog) is fetched exclusively through the FPI GraphQL client (`fdk-store`). Never use raw `fetch`, `axios`, or XHR to call Fynd APIs directly.

## Accessing the FPI Client

```jsx
import { useFPI } from "fdk-core/utils";
import { useGlobalStore } from "fdk-core/utils";

function ProductCard({ slug }) {
  const fpi = useFPI();

  // Read from store — reactively updates when store changes
  const product = useGlobalStore(fpi.getters.PRODUCT_DETAILS);
  const cartItems = useGlobalStore(fpi.getters.CART_ITEMS);
}
```

## Parallel Dispatch for Independent Fetches

Never chain independent FPI calls sequentially. Use `Promise.all()`.

**Incorrect (sequential waterfall — 3× latency):**

```jsx
useEffect(() => {
  async function load() {
    await fpi.catalog.getProduct({ slug });
    await fpi.catalog.getSimilarProducts({ slug });
    await fpi.catalog.getProductReviews({ slug });
  }
  load();
}, [slug]);
```

**Correct (parallel — 1× latency):**

```jsx
useEffect(() => {
  async function load() {
    await Promise.all([
      fpi.catalog.getProduct({ slug }),
      fpi.catalog.getSimilarProducts({ slug }),
      fpi.catalog.getProductReviews({ slug }),
    ]);
  }
  load();
}, [slug]);
```

## Get Only What You Need

GraphQL lets you request exact fields. Over-fetching wastes bandwidth and slows SSR.

```jsx
// BAD — fetches entire product object
fpi.catalog.getProduct({ slug });

// GOOD — pass only the fields your component renders
fpi.catalog.getProduct({ slug, fields: ["name", "price", "media", "slug"] });
```

## Defer Store Reads to Usage Point

Don't subscribe to store values that are only needed inside callbacks. Read them lazily.

**Incorrect (subscribes globally, causes re-renders on every cart change):**

```jsx
function AddToCartButton({ productId }) {
  const fpi = useFPI();
  const cart = useGlobalStore(fpi.getters.CART); // re-renders on every cart update

  function handleAdd() {
    if (!cart.loading) {
      fpi.cart.addItems({ productId });
    }
  }
}
```

**Correct (reads store value only when needed):**

```jsx
function AddToCartButton({ productId }) {
  const fpi = useFPI();

  function handleAdd() {
    const cart = fpi.store.getState(fpi.getters.CART); // read on demand
    if (!cart.loading) {
      fpi.cart.addItems({ productId });
    }
  }
}
```

## SWR Cache Layer

When `ENABLE_SWR_CACHE` is set in theme settings, the SWR wrapper in `theme/helper/fpi-swr-wrapper.js` intercepts FPI reads and serves cached responses. The cache:
- Holds up to 100 entries, max 5MB total
- Revalidates in the background on window focus and network reconnect
- Uses LRU eviction

Do not build a separate caching layer. If you need to control caching behavior, configure the SWR wrapper options.

## Data Resolver Responsibilities

| Resolver | File | When it runs | Purpose |
|----------|------|-------------|---------|
| `globalDataResolver` | `theme/helper/lib.js` | Once on app init (SSR) | Fetch global config, location, user session |
| `pageDataResolver` | `theme/helper/lib.js` | Once per page (SSR) | Fetch page-specific initial data |

Keep resolvers lean — they block the SSR render. Defer non-critical data to `useEffect`.
