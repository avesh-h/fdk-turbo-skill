---
name: fynd-theme
description: Context and best practices for working on the Fynd FDK React Theme — a production e-commerce theme connected to the Fynd Commerce platform. Use this skill when writing, reviewing, or refactoring any code in this repository. Covers FPI/FDK-specific patterns, SSR compatibility, GraphQL data fetching, image optimization, analytics events, and platform integration rules.
license: ISC
metadata:
  author: fynd
  version: "2.0.0"
---

# Fynd FDK React Theme Skill

Production-grade React e-commerce theme tightly coupled to the Fynd Commerce platform. All logic for payments, checkout, cart, shipping, delivery, OMS, post-order journey, and authentication is **already implemented** in the theme's hooks, sections, and pages. Do not re-implement existing platform modules — extend or modify them.

## Execution Visibility

When this skill is used, begin the response with: `Executing the fynd-theme skill.`

---

## Platform Context

This is **not a standalone React app**. It runs inside the Fynd platform runtime, which handles:
- Routing and page rendering (SSR + SPA hydration)
- All e-commerce modules — cart, checkout, payments, OMS, auth — already wired via FDK hooks
- Theme configuration surfaced in the Fynd Admin Panel via `settings_schema.json`
- Asset delivery and deployment via FDK-CLI (`fdk theme serve`, `fdk theme sync`)

**Four content layers:**

| Layer | Path | Role |
|-------|------|------|
| Pages | `theme/pages/` | Full route-level components (45 pages) |
| Sections | `theme/sections/` | Reusable content blocks (58 sections); primary customization units merchants drag-and-drop |
| Components | `theme/components/` | Primitive UI components (47) used by pages and sections |
| Page Layouts | `theme/page-layouts/` | Heavy logic hooks per feature area (checkout, cart, PDP, auth, orders, etc.) — do not overwrite |

---

## Rule Categories by Priority

| Priority | Category | Impact |
|----------|----------|--------|
| 0 | Never Overwrite Existing Logic Hooks | CRITICAL |
| 1 | SSR Compatibility | CRITICAL |
| 2 | FPI Data Fetching | CRITICAL |
| 3 | Platform Boundaries | CRITICAL |
| 4 | URL & Navigation | HIGH |
| 5 | Analytics & Events | HIGH |
| 6 | Image Optimization | HIGH |
| 7 | State & Re-render Optimization | MEDIUM |
| 8 | Styling Conventions | MEDIUM |
| 9 | Component Design | MEDIUM |
| 10 | Theme Configuration | MEDIUM |
| 11 | Bundle & Performance | MEDIUM |

---

## 0. Never Overwrite Existing Logic Hooks (CRITICAL)

> **This is the single most important rule for this codebase.**

The theme contains complete, production-tested implementations of every major e-commerce operation — payments, authentication, cart, address management, checkout, OMS, wishlist, and more. These live across two directories:

**`theme/helper/hooks/`** — FPI-integrated hooks (20+ files):

| Hook file | What it owns |
|-----------|-------------|
| `useAddress.jsx` | Fetch, add, update, remove addresses; pincode locality lookup |
| `useThemeConfig.jsx` | Global and page-level theme config access |
| `useWishlist.jsx` | Wishlist add/remove/toggle |
| `usePolling.jsx` | Background polling logic |
| `useLocalStorage.jsx` | SSR-safe localStorage wrapper |
| `useWindowWidth.jsx` | Responsive breakpoint detection |
| `hooks.jsx` | Barrel export — `useSnackbar`, and other shared utilities |
| *(and more)* | auth utilities, delivery promise, form schema, etc. |

**`theme/page-layouts/`** — Heavy feature-level hooks (~30 feature areas):

| Path | What it owns |
|------|-------------|
| `single-checkout/payment/usePayment.jsx` | Full payment flow: CARD, UPI, QR, NB, WL, COD, EMI, Pay Later, credit note, partial payment, payment links |
| `cart/useCart.jsx` | Cart CRUD, coupon logic, cart breakup |
| `pdp/` | Product detail, size selection, pincode check |
| `auth/` | Login, register, OTP, social auth |
| `orders/` | Order list, order detail, return/cancel/refund |
| `plp/` | Product listing, filters, pagination |
| `address/` | Address page logic |
| *(and more)* | Every page area has its own dedicated hook file |

### The Rule

**NEVER overwrite, replace, or rewrite any of these hook files.** They contain:
- Hundreds of edge cases built from real production bugs
- Platform state sync contracts that break silently when disrupted
- Integrated analytics dispatch tied to exact user action timing
- SSR-safe patterns that took significant effort to stabilize

**What you CAN do:**
- Add new exported functions to an existing hook file (append only)
- Add new state variables for new UI behaviour
- Call existing hook functions from new components
- Create a new hook file in `theme/helper/hooks/` for genuinely new functionality

**What you must NEVER do:**
```jsx
// NEVER — replaces the entire production payment flow
const usePayment = (fpi) => {
  // rewriting from scratch...
};
export default usePayment;

// NEVER — replaces production address logic
export const useAddress = ({ fpi }) => {
  // rewriting from scratch...
};
```

**Before touching any hook file: read the entire file first.** If a task requires modifying an existing hook, explain precisely which lines change and why, and confirm it does not remove any existing exported function or state.

---

## 1. SSR Compatibility (CRITICAL)

All code executes on the server during initial page render. Any access to browser globals at module or component initialization time will crash the server render.

**Never access at module scope or component body:**

```jsx
// BAD — crashes server render
const isMobile = window.innerWidth < 768;

function MyComponent() {
  const token = localStorage.getItem("auth_token"); // crashes on server
  return <div />;
}
```

**Always guard or move to effects/handlers:**

```jsx
// GOOD — guarded inline
const width = typeof window !== "undefined" ? window.innerWidth : 0;

// GOOD — effect is browser-only
function MyComponent() {
  const [isMobile, setIsMobile] = useState(false);
  useEffect(() => {
    setIsMobile(window.innerWidth < 768);
    const token = localStorage.getItem("auth_token");
  }, []);
}
```

**Rules:**
- Never use `window`, `document`, `navigator`, `location` outside effects or event handlers
- Never use `localStorage`/`sessionStorage`/`IntersectionObserver`/`ResizeObserver` outside `useEffect`
- Event handlers are browser-only by nature — safe to use browser APIs inside them

**Every page needs two data fetch paths:**

```jsx
// SSR path — in pageDataResolver (theme/helper/lib.js), runs on server
// SPA path — in useEffect, runs when navigating client-side
useEffect(() => {
  if (!productData) {
    fpi.catalog.getProduct({ slug });
  }
}, [slug]);
```

Both must exist. The `pageDataResolver` handles SSR; `useEffect` handles SPA navigation.

> Deep dive: `fynd-theme/rules/ssr-browser-guards.md`

---

## 2. FPI Data Fetching (CRITICAL)

All platform data is fetched through the FPI GraphQL client (`fdk-store`). Never use raw `fetch` or `axios` to call Fynd APIs.

**Accessing the client:**

```jsx
import { useFPI, useGlobalStore } from "fdk-core/utils";

function ProductCard({ slug }) {
  const fpi = useFPI();
  const product = useGlobalStore(fpi.getters.PRODUCT_DETAILS);
  const cartItems = useGlobalStore(fpi.getters.CART_ITEMS);
}
```

**Parallel dispatch for independent fetches — never sequential:**

```jsx
// BAD — 3× latency waterfall
await fpi.catalog.getProduct({ slug });
await fpi.catalog.getSimilarProducts({ slug });
await fpi.catalog.getProductReviews({ slug });

// GOOD — single round-trip
await Promise.all([
  fpi.catalog.getProduct({ slug }),
  fpi.catalog.getSimilarProducts({ slug }),
  fpi.catalog.getProductReviews({ slug }),
]);
```

**Get only what you need in a single call. Avoid over-fetching.**

**Defer store reads to usage point — avoid subscribing to values only needed in callbacks:**

```jsx
// BAD — subscribes globally, re-renders on every cart update
const cart = useGlobalStore(fpi.getters.CART);
function handleAdd() { if (!cart.loading) fpi.cart.addItems(...); }

// GOOD — reads on demand, no subscription
function handleAdd() {
  const cart = fpi.store.getState(fpi.getters.CART);
  if (!cart.loading) fpi.cart.addItems(...);
}
```

**SWR caching:** When `ENABLE_SWR_CACHE` is set in theme settings, use `theme/helper/fpi-swr-wrapper.js`. Do not build a separate cache layer.

**Data resolver responsibilities:**

| Resolver | File | When | Purpose |
|----------|------|------|---------|
| `globalDataResolver` | `theme/helper/lib.js` | Once on app init (SSR) | Global config, location, user session |
| `pageDataResolver` | `theme/helper/lib.js` | Per page (SSR) | Page-specific initial data |

Keep both lean — they block the SSR render. Defer non-critical data to `useEffect`.

> Deep dive: `fynd-theme/rules/fpi-data-fetching.md`

---

## 3. Platform Boundaries — Don't Reimplement Built-in Modules (CRITICAL)

The platform provides complete implementations of every core e-commerce module. Do not recreate them.

**Modules already implemented — do not recreate:**

| Module | Handled by |
|--------|-----------|
| Authentication (login, register, OTP, social) | FPI store + `theme/helper/auth-guard.js` |
| Cart (add, remove, update, coupons) | `fpi.cart.*` actions |
| Checkout (address, delivery, payment) | `theme/sections/checkout.jsx` + FPI actions |
| Payments | FPI payment actions + platform PG integration |
| Order placement & OMS | `fpi.order.*` actions |
| Post-order (returns, cancellations, refunds) | FPI order management actions |
| Shipping / delivery promise | FPI logistics actions |
| Wishlist | `fpi.catalog.wishlist.*` actions |
| Product catalog, search, filters | `fpi.catalog.*` actions |
| User profile | `fpi.user.*` actions |

**Incorrect — parallel custom cart state:**

```jsx
// Creates conflicting state — diverges from platform cart
const [cartItems, setCartItems] = useState([]);
function addToCart(product) {
  setCartItems((prev) => [...prev, product]);
  fetch("/api/cart/add", { body: JSON.stringify(product) }); // bypasses FPI
}
```

**Correct — use FPI:**

```jsx
const fpi = useFPI();
const cartItems = useGlobalStore(fpi.getters.CART_ITEMS);

async function addToCart(product, size) {
  await fpi.cart.addItems({
    items: [{ item_id: product.uid, item_size: size, quantity: 1 }],
  });
  // store auto-updates — no manual setState
}
```

**Before modifying any file in `theme/helper/hooks/`, `theme/page-layouts/`, or `theme/sections/`, read the entire file first.** These files manage analytics dispatch, platform state sync, error handling, SSR safety, and production edge cases accumulated over many releases. Partial rewrites silently break these invariants.

**The golden rule: extend, don't replace.** Add new functions and state. Never remove or rewrite existing exports.

> Deep dive: `fynd-theme/rules/platform-boundaries.md`

---

## 4. URL & Navigation — Never Hardcode Storefront URLs (HIGH)

The platform appends UTM parameters and manages multi-tenant routing. Hardcoded paths strip attribution and break across deployments.

**Incorrect:**

```jsx
router.push("/products/red-shoes");       // strips UTM
window.location.href = "/cart";           // bypasses platform router
<a href="/collection/sale">Shop Sale</a>  // breaks multi-tenant
```

**Correct — action objects:**

```jsx
import { useNavigate } from "fdk-core/utils";
import { FDKLink } from "fdk-core/components";

const navigate = useNavigate();
navigate({ action: "product", params: { slug: product.slug } });

<FDKLink action="collection" params={{ slug: "sale" }}>Shop Sale</FDKLink>
```

**Common actions:**

| Page | Action |
|------|--------|
| Home | `{ action: "home" }` |
| PDP | `{ action: "product", params: { slug } }` |
| Collection / PLP | `{ action: "collection", params: { slug } }` |
| Cart | `{ action: "cart" }` |
| Checkout | `{ action: "checkout" }` |
| Order detail | `{ action: "order-detail", params: { orderId } }` |
| Profile | `{ action: "profile" }` |
| Search | `{ action: "search", params: { query } }` |

Never hardcode domain names. For full URLs: `fpi.router.getPageURL({ action, params })`.

> Deep dive: `fynd-theme/rules/url-navigation.md`

---

## 5. Analytics & FPI Events (HIGH)

FPI analytics events feed merchant dashboards, GTM, and attribution. Incorrect events corrupt reporting.

**Fire events only at their intended trigger point:**

| Event | Correct Trigger |
|-------|----------------|
| `product_viewed` | When PDP data first loads |
| `add_to_cart` | After successful `fpi.cart.addItems()` response |
| `checkout_started` | When user lands on checkout page |
| `purchase` | After successful order confirmation |
| `product_list_viewed` | When PLP renders with its product list |

**Never fire events in effects that can re-run:**

```jsx
// BAD — fires on every re-render
useEffect(() => {
  fpi.analytics.fireEvent("product_viewed", { product });
});

// GOOD — fires exactly once when product data loads
const hasTracked = useRef(false);
useEffect(() => {
  if (product && !hasTracked.current) {
    hasTracked.current = true;
    fpi.analytics.fireEvent("product_viewed", { product });
  }
}, [product]);
```

**Never fire events during SSR** — analytics require browser context (cookies, GTM).

**Don't push to `window.dataLayer` directly** — use `fpi.analytics.fireEvent()`.

**Before adding a new event call, check if the existing hook already fires it.**

> Deep dive: `fynd-theme/rules/analytics-events.md`

---

## 6. Image Optimization (HIGH)

Always resize images through Pixelbin via `transformImage`. Never use raw CDN URLs.

```jsx
import { transformImage } from "../../helper/utils";

// Actual signature: transformImage(url, width)
// Size to actual rendered dimensions
<img
  src={transformImage(product.media[0].url, 400)}
  width={400}
  height={400}
  loading="lazy"
  alt={product.name}
/>
```

**LCP images (hero, first product card) — eager + high priority:**

```jsx
<img
  src={transformImage(src, 800)}
  loading="eager"
  fetchpriority="high"
  alt={alt}
/>
```

**Rules:**
- Always set `width`/`height` attributes — prevents layout shift (CLS)
- Request images at their rendered size — not original resolution
- `loading="lazy"` for all below-fold images
- Use `srcSet` for components that render at different sizes across breakpoints

> Deep dive: `fynd-theme/rules/image-optimization.md`

---

## 7. State & Re-render Optimization (MEDIUM)

**Hoist non-primitive default props outside components:**

```jsx
// BAD — new object reference every render, breaks memo
<Component style={{ padding: 8 }} />

// GOOD
const DEFAULT_STYLE = { padding: 8 };
<Component style={DEFAULT_STYLE} />
```

**Use functional setState when new state derives from previous:**

```jsx
setCount((prev) => prev + 1); // not setCount(count + 1)
```

**Use `useRef` for transient values that don't need re-renders:**
Scroll position, timer IDs, animation frame IDs, "has tracked" flags.

**Use `React.memo` on list items:**
Product cards, order rows, review items — any component rendered in a `.map()`.

**Prefer `startTransition` for non-urgent updates:**
Search filtering, tab switching, sort changes.

**Derive state during render instead of syncing with `useEffect`:**

```jsx
// BAD — effect + state for derived value
const [isDiscounted, setIsDiscounted] = useState(false);
useEffect(() => { setIsDiscounted(price < originalPrice); }, [price]);

// GOOD — compute directly
const isDiscounted = price < originalPrice;
```

Relevant Vercel rules: `rerender-memo.md`, `rerender-memo-with-default-value.md`, `rerender-functional-setstate.md`, `rerender-use-ref-transient-values.md`, `rerender-derived-state-no-effect.md`, `rerender-transitions.md`

---

## 8. Styling Conventions (MEDIUM)

- **Less modules** (`.module.less`) for all component-scoped styles — no `styled-components`, `emotion`, or Tailwind
- **CSS variables** for merchant-configurable values (colors, fonts, spacing) — set by `ThemeProvider`, consumed in Less
- **`theme/styles/base.global.less`** for global base styles only (resets, `:root` variable declarations, third-party overrides)
- **Mobile-first** responsive styles via `@media (min-width: ...)` breakpoints
- **Inline `style` props only for truly dynamic JS-computed values** (cursor position, measured heights) — not for layout/design

```less
// GOOD — Less module with CSS variables
.button {
  background-color: var(--button-primary-bg);
  color: var(--button-primary-text);
  font-family: var(--font-body);

  @media (min-width: 768px) {
    padding: 12px 24px;
  }
}
```

> Deep dive: `fynd-theme/rules/styling-conventions.md`

---

## 9. Component Design (MEDIUM)

**Check before creating** — these already exist:

| Need | Where to look |
|------|--------------|
| Custom hooks | `theme/helper/hooks/` directory — `useThemeConfig`, `useAddress`, `usePolling`, `useWishlist`, etc. (20+ files) |
| Feature logic hooks | `theme/page-layouts/` — `usePayment`, `useCart`, PDP/auth/orders logic |
| Utilities | `theme/helper/utils.js` — `transformImage(url, width)`, `sanitizeHTMLTag`, color utils |
| UI primitives | `theme/components/` — carousel, modal, loader, form fields, rating, breadcrumb |
| Data resolvers | `theme/helper/lib.js` |

**Keep components under ~300 lines.** Split by responsibility when they grow.

**Don't prop-drill beyond 2 levels.** Use `useGlobalStore` or context.

**Sanitize all platform HTML** — product descriptions, CMS content, banners:

```jsx
import { sanitizeHTMLTag } from "../../helper/utils";
import parse from "html-react-parser";

// With sanitization (never skip this)
<div>{parse(sanitizeHTMLTag(product.description))}</div>

// Never:
<div dangerouslySetInnerHTML={{ __html: product.description }} />
```

**Hoist static JSX outside component body:**

```jsx
// BAD — new reference every render
function Icon() { return <svg>...</svg>; }
function Button() { return <button><Icon /></button>; }

// GOOD — created once
const ICON = <svg>...</svg>;
function Button() { return <button>{ICON}</button>; }
```

**Sections vs Components:**
- **Sections** — connected to FPI store, declare their own `props` schema, work standalone
- **Components** — presentational, receive data via props, reusable across sections

> Deep dive: `fynd-theme/rules/component-design.md`

---

## 10. Theme Configuration (MEDIUM)

Any design decision a merchant should control must be in `settings_schema.json` — never hardcoded.

**Read config via `useThemeConfig()` hook:**

```jsx
import { useThemeConfig } from "../../helper/hooks";

function MyComponent() {
  const { global_config, page_config } = useThemeConfig();
  const primaryFont = global_config?.custom?.props?.font_body;
  const showReviews = page_config?.props?.show_reviews?.value;
}
```

**Section props are received directly:**

```jsx
function HeroBanner({ props, globalConfig }) {
  const { heading, background_color, cta_text } = props;
  return <div style={{ backgroundColor: background_color?.value }}>{heading?.value}</div>;
}
```

**Schema types:** `font`, `range`, `select`, `checkbox`, `text`, `url`, `image_picker`, `color`, `extension`

**Use CSS variables for runtime-configurable values** — set by `ThemeProvider` on `:root`, consumed in Less. Never inline dynamic JS config values as style props.

> Deep dive: `fynd-theme/rules/theme-configuration.md`

---

## 11. Bundle & Performance (MEDIUM)

**Every page import in `theme/index.jsx` must be a dynamic import with `webpackChunkName`:**

```jsx
// GOOD — code split per page
getProductDescription: () =>
  import(/* webpackChunkName: "product-description" */ "./pages/product-description"),
```

**Import from specific module paths, not barrel index files:**

```jsx
// BAD — may load entire library
import { formatPrice } from "../../helper";

// GOOD
import { formatPrice } from "../../helper/utils";
```

**Defer third-party scripts after hydration:**

```jsx
useEffect(() => {
  // Copilot, Google Maps, analytics — load after first render
  import("./copilot").then((m) => m.init());
}, []);
```

**For scroll-heavy pages** (PLP, order history) consider `content-visibility: auto` in Less for off-screen sections.

Relevant Vercel rules: `bundle-barrel-imports.md`, `bundle-dynamic-imports.md`, `bundle-defer-third-party.md`, `rendering-content-visibility.md`

---

## Not Applicable to This Codebase

**`frontend-design` skill** — for building new UIs with bold aesthetic choices. This codebase has an established design system (Less modules, CSS variables, `settings_schema.json`). Do not apply those aesthetic guidelines here.

**Vercel `server-*` rules** (RSC, `React.cache()`, `after()`) — these are Next.js App Router patterns. This project uses a custom Webpack + FDK SSR runtime. Use FPI data resolvers for server-side patterns instead.

---

## Key File Reference

| File | Purpose |
|------|---------|
| `theme/index.jsx` | Entry point — initializes FPI, exports all page/section/component references |
| `theme/providers/global-provider.jsx` | ThemeProvider — wraps app, SEO meta, Copilot.live init |
| `theme/helper/utils.js` | `transformImage(url, width)`, `sanitizeHTMLTag`, color utils |
| `theme/helper/hooks/` | Directory of 20+ custom hooks: `useThemeConfig`, `useAddress`, `usePolling`, `useWishlist`, etc. |
| `theme/helper/hooks/index.jsx` | Barrel export for all hooks in the directory |
| `theme/page-layouts/` | Feature-level logic hooks: `usePayment`, `useCart`, PDP, auth, orders (do not overwrite) |
| `theme/helper/fpi-swr-wrapper.js` | SWR caching layer for FPI GraphQL responses |
| `theme/helper/lib.js` | `globalDataResolver`, `pageDataResolver` |
| `theme/helper/auth-guard.js` | Auth guard utilities |
| `settings_schema.json` | All merchant-configurable theme props |
| `webpack.config.js` | Build config — Less modules, code splitting, polyfills |
| `copilot/index.js` | Copilot.live AI integration entry |

---

## Extended Rule Files (Deep Dives)

For more detailed explanations with additional code examples on each topic:

| Fynd Rule File | Topic |
|----------------|-------|
| `fynd-theme/rules/ssr-browser-guards.md` | SSR safety patterns |
| `fynd-theme/rules/fpi-data-fetching.md` | FPI GraphQL patterns |
| `fynd-theme/rules/platform-boundaries.md` | Module ownership boundaries |
| `fynd-theme/rules/url-navigation.md` | Action-based navigation |
| `fynd-theme/rules/analytics-events.md` | Event dispatch safety |
| `fynd-theme/rules/image-optimization.md` | Pixelbin/transformImage usage |
| `fynd-theme/rules/theme-configuration.md` | settings_schema patterns |
| `fynd-theme/rules/styling-conventions.md` | Less modules + CSS variables |
| `fynd-theme/rules/component-design.md` | Component patterns |

Vercel React rules (generic React/JS performance): `vercel-react-best-practices/rules/`
Most relevant: `async-parallel.md`, `bundle-barrel-imports.md`, `bundle-dynamic-imports.md`, `rerender-memo.md`, `rerender-derived-state-no-effect.md`, `rendering-hoist-jsx.md`, `rendering-conditional-render.md`
