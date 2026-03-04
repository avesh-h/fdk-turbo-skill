---
title: Image Optimization with transformImage
impact: HIGH
impactDescription: Unoptimized images are the #1 cause of slow LCP on e-commerce pages
tags: images, pixelbin, transformImage, performance, lcp
---

## Image Optimization with transformImage

All product images, banners, and catalog media flow through Pixelbin's CDN. Always use the `transformImage` utility to resize images to their actual rendered dimensions before rendering — never use raw CDN URLs directly.

## Using transformImage

```jsx
import { transformImage } from "../../helper/utils";

function ProductCard({ product }) {
  return (
    <img
      src={transformImage({
        src: product.media[0].url,
        width: 400,   // match rendered width
        height: 400,  // match rendered height
        format: "webp",
      })}
      alt={product.name}
      loading="lazy"
      width={400}
      height={400}
    />
  );
}
```

**Incorrect (raw URL — loads full resolution, no format optimization):**

```jsx
<img src={product.media[0].url} alt={product.name} />
```

## Rules

### Size to Rendered Dimensions
Request the image at the size it will be displayed. For responsive components, compute size from container width:

```jsx
function BannerImage({ src, containerWidth }) {
  // For full-width banners, use viewport width
  const width = containerWidth || (typeof window !== "undefined" ? window.innerWidth : 1200);

  return (
    <img
      src={transformImage({ src, width, format: "webp" })}
      alt=""
      loading="eager"
      fetchpriority="high"
    />
  );
}
```

### LCP Images — Eager + High Priority
The hero image, first product card image, and above-fold banners are LCP candidates. Load them eagerly:

```jsx
// Hero / first product — LCP candidate
<img
  src={transformImage({ src, width: 800, format: "webp" })}
  loading="eager"
  fetchpriority="high"
  alt={alt}
/>
```

### Below-Fold Images — Lazy
All other images use lazy loading:

```jsx
<img
  src={transformImage({ src, width: 400, format: "webp" })}
  loading="lazy"
  alt={alt}
/>
```

### Always Set width/height Attributes
Prevents layout shift (CLS). Set them to the rendered dimensions so the browser reserves space:

```jsx
<img
  src={transformImage({ src, width: 300, height: 300 })}
  width={300}
  height={300}
  loading="lazy"
  alt={alt}
/>
```

### Responsive Images — Use srcSet
For components that render at different sizes across breakpoints:

```jsx
const src = product.media[0].url;
<img
  src={transformImage({ src, width: 400 })}
  srcSet={`
    ${transformImage({ src, width: 300 })} 300w,
    ${transformImage({ src, width: 600 })} 600w,
    ${transformImage({ src, width: 900 })} 900w
  `}
  sizes="(max-width: 768px) 100vw, 50vw"
  loading="lazy"
  alt={alt}
/>
```
