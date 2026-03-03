---
title: Theme Configuration & settings_schema.json
impact: MEDIUM
impactDescription: Hardcoded design decisions block merchants from customizing their storefront
tags: theme-config, settings-schema, merchant-customization, css-variables
---

## Theme Configuration & settings_schema.json

Any design decision a merchant should be able to change (color, font, layout toggle, spacing, feature flag) must be exposed through `settings_schema.json` — never hardcoded in components or Less files.

## Where Configs Live

- **`settings_schema.json`** (root) — global theme-level settings (fonts, header layout, cart options, product card styles)
- **Section `props` schema** — section-level settings declared inline alongside each section component
- **`config.json`** — default font configuration (Poppins)

## Reading Theme Config in Components

Always use the `useThemeConfig()` hook from `theme/helper/hooks.jsx`. Never access raw config objects directly.

**Incorrect:**

```jsx
// Don't access window.__theme_config__ directly
const primaryColor = window.__theme_config__.colors.primary;

// Don't import config JSON and read it statically
import config from "../../settings_schema.json";
```

**Correct:**

```jsx
import { useThemeConfig } from "../../helper/hooks";

function MyComponent() {
  const { global_config, page_config } = useThemeConfig();
  const primaryFont = global_config?.custom?.props?.font_body;
  const showReviews = page_config?.props?.show_reviews?.value;
}
```

## Reading Section Props

Section components receive their configured props directly via the `props` prop:

```jsx
function HeroBanner({ props, blocks, globalConfig }) {
  const { heading, subheading, cta_text, background_color } = props;
  return (
    <div style={{ backgroundColor: background_color?.value }}>
      <h1>{heading?.value}</h1>
    </div>
  );
}
```

## Adding New Configurable Options

When adding a new option merchants should control, add it to the schema first:

```json
// In settings_schema.json or section props
{
  "id": "show_size_guide",
  "label": "Show Size Guide",
  "type": "checkbox",
  "default": true,
  "info": "Display size guide link on product pages"
}
```

Available schema types: `font`, `range`, `select`, `checkbox`, `text`, `url`, `image_picker`, `color`, `extension`

## Use CSS Variables for Dynamic Theming

For values that change at runtime based on config (colors, fonts, spacing), set CSS variables in the `ThemeProvider` and consume them in Less — never inline the JS values into style props.

**Incorrect (breaks theming, causes re-renders):**

```jsx
<div style={{ color: themeConfig.primaryColor }}>
```

**Correct (CSS variable set by ThemeProvider, consumed in Less):**

```less
// component.module.less
.heading {
  color: var(--primary-color);
  font-family: var(--font-body);
}
```

```jsx
// ThemeProvider sets these once
document.documentElement.style.setProperty("--primary-color", config.primaryColor);
```

## Feature Flags via Theme Settings

Use `checkbox` type settings to gate features. Read via `useThemeConfig()`:

```jsx
const { global_config } = useThemeConfig();
const isCopilotEnabled = global_config?.custom?.props?.storefront_copilot_actions?.value;

if (isCopilotEnabled) {
  initCopilot();
}
```
