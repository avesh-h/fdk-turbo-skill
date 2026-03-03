---
title: URL & Navigation — Never Hardcode Storefront URLs
impact: HIGH
impactDescription: Hardcoded URLs strip UTM parameters, break analytics, and fail across multi-tenant storefronts
tags: navigation, urls, utm, analytics, routing, fdk
---

## URL & Navigation — Never Hardcode Storefront URLs

The Fynd platform appends UTM parameters, handles multi-tenant routing, and manages storefront base paths dynamically. Hardcoding any path or domain breaks attribution tracking, search ranking, and multi-tenant deployments.

## Use Action Objects for Navigation

**Incorrect (hardcoded path — strips all UTM parameters):**

```jsx
// BAD
router.push("/products/red-shoes");
window.location.href = "/cart";
<a href="/collection/sale">Shop Sale</a>
```

**Correct (action-based navigation — preserves UTM + platform routing):**

```jsx
import { useNavigate } from "fdk-core/utils";

function ProductLink({ product }) {
  const navigate = useNavigate();

  function handleClick() {
    navigate({
      action: "product",
      params: { slug: product.slug },
    });
  }

  return <button onClick={handleClick}>{product.name}</button>;
}
```

**Correct (using FDK link utility for anchor tags):**

```jsx
import { FDKLink } from "fdk-core/components";

<FDKLink action="product" params={{ slug: product.slug }}>
  {product.name}
</FDKLink>
```

## Common Action Types

| Page | Action |
|------|--------|
| Home | `{ action: "home" }` |
| PDP | `{ action: "product", params: { slug } }` |
| PLP / Collection | `{ action: "collection", params: { slug } }` |
| Cart | `{ action: "cart" }` |
| Checkout | `{ action: "checkout" }` |
| Order Detail | `{ action: "order-detail", params: { orderId } }` |
| Profile | `{ action: "profile" }` |
| Login | `{ action: "login" }` |
| Search | `{ action: "search", params: { query } }` |
| Category | `{ action: "category", params: { slug } }` |

For static pages (About, Privacy, FAQs), use the action definitions from platform config — never hardcode their paths.

## No Hardcoded Domain Names

**Incorrect:**

```jsx
const url = `https://my-store.fynd.com/products/${slug}`;
fetch("https://api.fynd.com/service/..."); // use FPI instead
```

**Correct:**

```jsx
// Let the platform construct the full URL
const url = fpi.router.getPageURL({ action: "product", params: { slug } });
```

## Query Parameters for Search/Filter State

Search and filter state must live in URL query params (for SEO and shareability). Use the platform's URL utilities — do not manually construct query strings.

```jsx
// BAD
const newUrl = `/products?category=${cat}&price=${min}-${max}`;

// GOOD — use fdk router utilities that handle encoding + UTM preservation
fpi.router.updateQuery({ category: cat, price: `${min}-${max}` });
```
