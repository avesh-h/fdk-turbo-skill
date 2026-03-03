---
title: Platform Boundaries — Don't Reimplement Built-in Modules
impact: CRITICAL
impactDescription: Reimplementing platform modules creates conflicting state, breaks analytics, and duplicates thousands of lines of tested logic
tags: platform, fdk, cart, checkout, payments, auth, oms
---

## Platform Boundaries — Don't Reimplement Built-in Modules

The Fynd platform provides complete, production-tested implementations of every core e-commerce module. These are wired through FPI store actions and hooks in `theme/helper/hooks.jsx`. Do not reimplement, replace, or bypass them.

## Modules Already Implemented — Do Not Recreate

| Module | Already Handled By |
|--------|-------------------|
| Authentication (login, register, OTP, social auth) | FPI store + `theme/helper/auth-guard.js` |
| Cart (add, remove, update quantity, apply coupons) | `fpi.cart.*` actions |
| Checkout flow (address, delivery, payment selection) | `theme/sections/checkout.jsx` + FPI actions |
| Payment processing | FPI payment actions + platform PG integration |
| Order placement | FPI order actions |
| OMS / Order tracking | `fpi.order.*` actions |
| Post-order journey (returns, cancellations, refunds) | FPI order management actions |
| Shipping / Delivery promise calculation | FPI logistics actions |
| Wishlist | `fpi.catalog.wishlist.*` actions |
| Product catalog (listing, search, filters, PDP) | `fpi.catalog.*` actions |
| User profile management | `fpi.user.*` actions |
| Coupons & offers | FPI cart coupon actions |
| Pincode serviceability check | FPI logistics actions |

## What "Don't Reimplement" Means

**Incorrect — building a custom cart state:**

```jsx
// DON'T do this — creates parallel state that conflicts with FPI cart
const [cartItems, setCartItems] = useState([]);

function addToCart(product) {
  setCartItems((prev) => [...prev, product]); // diverges from platform cart
  fetch("/api/cart/add", { body: JSON.stringify(product) }); // bypasses FPI
}
```

**Correct — use FPI cart actions:**

```jsx
const fpi = useFPI();
const cartItems = useGlobalStore(fpi.getters.CART_ITEMS);

async function addToCart(product, size) {
  await fpi.cart.addItems({
    items: [{ item_id: product.uid, item_size: size, quantity: 1 }],
  });
  // Cart state auto-updates in the store — no manual setState needed
}
```

## Extend, Don't Replace

If you need to customize module behavior, extend the existing implementation:

```jsx
// WRONG — replace the existing checkout section
function MyCustomCheckout() { /* full reimplementation */ }

// RIGHT — extend the existing section with additional UI
function CheckoutWithGiftNote() {
  return (
    <>
      <CheckoutSection {...props} /> {/* existing — keep it */}
      <GiftNoteInput />              {/* your addition */}
    </>
  );
}
```

## Read Before Modifying

Before modifying any hook in `theme/helper/hooks.jsx` or any section in `theme/sections/`, read the full file. These hooks manage:
- Analytics event dispatch
- Platform state synchronization
- Error handling with platform-specific error codes
- Edge cases accumulated from production use

Partial rewrites break these invariants.

## When You Genuinely Need Custom Logic

If platform actions don't cover your use case:
1. Check `fpi` client API for less-obvious action methods
2. Check if a `theme/helper/hooks.jsx` hook already handles it
3. Check existing section implementations in `theme/sections/`
4. Only then add new logic — and add it *alongside* existing platform calls, not replacing them
