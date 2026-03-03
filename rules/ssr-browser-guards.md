---
title: SSR Browser Guards
impact: CRITICAL
impactDescription: Prevents server crash on first render — code runs in Node.js before hydration
tags: ssr, window, document, server-rendering, hydration
---

## SSR Browser Guards

All theme code executes on the server during initial page render. Any access to browser globals (`window`, `document`, `navigator`, `localStorage`, `sessionStorage`) at module or component initialization time will crash the server render.

**Incorrect (crashes server):**

```jsx
// module scope — runs on server immediately
const isMobile = window.innerWidth < 768;

function MyComponent() {
  // component body runs on server — crashes
  const token = localStorage.getItem("auth_token");
  return <div>{isMobile ? "mobile" : "desktop"}</div>;
}
```

**Correct (guarded access):**

```jsx
function MyComponent() {
  const [isMobile, setIsMobile] = useState(false);

  useEffect(() => {
    // useEffect only runs in browser — safe
    setIsMobile(window.innerWidth < 768);
    const token = localStorage.getItem("auth_token");
  }, []);

  return <div>{isMobile ? "mobile" : "desktop"}</div>;
}
```

**Correct (inline guard for computed values):**

```jsx
const width = typeof window !== "undefined" ? window.innerWidth : 0;
```

**Correct (event handlers are browser-only by nature):**

```jsx
function handleScroll() {
  // Fine — event handlers never run on server
  const y = window.scrollY;
}
```

## Rules

1. Never access `window`, `document`, `navigator`, `location` at module scope or in component body (outside effects/handlers).
2. Never call `localStorage`/`sessionStorage` outside `useEffect` or event handlers.
3. Never use `IntersectionObserver`, `ResizeObserver`, `MutationObserver` outside `useEffect`.
4. For SSR-safe feature detection, use `typeof window !== "undefined"` guard.
5. For APIs needed on first render (e.g., theme color from localStorage), use an `<script>` inline tag pattern — see `rendering-hydration-no-flicker` in the Vercel rules.

## Every Page Needs Two Data Paths

Each page component must handle both render contexts:

```jsx
// SSR path — data resolved in pageDataResolver (theme/helper/lib.js)
// This runs on the server and populates the initial store state.

// SPA path — data fetched in useEffect when navigating client-side
useEffect(() => {
  if (!productData) {
    fpi.catalog.getProduct({ slug });
  }
}, [slug]);
```

The `pageDataResolver` handles the SSR path. The `useEffect` handles the SPA navigation path. Both must be present.
