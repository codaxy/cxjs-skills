# Styling and theming

## Who styles what

- **The theme styles the widgets** — fields, grids, buttons, windows. Change how widgets look through the theme (below), not by restyling them from the outside.
- **`mod` selects a widget variant**: `<Button mod="primary" />`, `mod="hollow"`, `mod="danger"`. It accepts an array: `mod={["stretch", "fixed"]}`. Each widget and theme defines its own mods.
- **Layout, spacing and custom elements** are styled with Tailwind in projects that use it, and with plain CSS or SCSS otherwise.
- **Inline `style`** is fine for one-off values — `style="width: 300px"` — though a Tailwind project should prefer utility classes.

## Classes

`class` and `className` behave the same: both accept a string or a structured object of `className: condition`, and both can be used together. **Prefer `class`.** Use them together to separate the static classes from the dynamic ones:

```tsx
<div class="rounded border p-2" />
<div class={{ "bg-yellow-100": truthy(m.$order.overdue), "text-gray-500": falsy(m.$order.active) }} />
<div className="rounded border p-2" class={{ "border-blue-500": truthy(m.selected) }} />
```

A structured `class` is better than a `computable` that builds a class string — each entry is evaluated on its own.

## The theme: cx-theme-variables

`cx-theme-variables` is the default theme for new projects. It is built on CSS custom properties, so it can be changed at runtime — including dark mode — without recompiling SCSS.

```ts
import "cx-theme-variables/dist/reset.css";
import "cx-theme-variables/dist/widgets.css";
import "cx-theme-variables/dist/charts.css";
import "cx-theme-variables/dist/svg.css";
import { defaultPreset, densityCompact, renderThemeVariables, roundingLarge } from "cx-theme-variables";

renderThemeVariables({ ...defaultPreset, ...roundingLarge, ...densityCompact });
```

- **Presets** set the whole palette: `defaultPreset`, `darkBluePreset`, `darkGrayPreset`, `oceanPreset`, … **Tweaks** adjust one aspect: `rounding*`, `density*`, `font*`.
- **Dark mode**: render a second preset under a media query — `renderThemeVariables(darkBluePreset, ":root", "@media (prefers-color-scheme: dark)")`.
- **Runtime switching**: `<ThemeVarsRoot theme={m.theme} />` renders the variables from the store; `<ThemeVarsDiv theme={…}>` scopes a theme to part of the page.
- **Single values** can be overridden in CSS: `:root { --cx-theme-primary-color: #1976d2; }`.

Preview presets and tweaks in the [Theme Editor](https://cxjs.io/themes).

## Tailwind

Load the CxJS theme into Tailwind's `components` layer, so that it ranks above Tailwind's base styles and below the utilities:

```css
@import "tailwindcss";

@layer components {
  @import "cx-theme-variables/dist/widgets.css";
}
```

For an SCSS theme, load it with `@include meta.load-css("…")` inside the layer instead — `@layer` does not combine with `@use`.

## Classic SCSS themes

Older projects use compile-time SCSS themes (`cx-theme-aquamarine`, `cx-theme-frost`, `cx-theme-material`, …). They are configured through SCSS variables, state style maps and CSS overrides — follow the project's existing setup and see [Classic Themes](https://cxjs.io/docs/concepts/classic-themes.md).

## Documentation

| Task | Page |
|---|---|
| choosing and installing a theme | [Themes](https://cxjs.io/docs/intro/themes.md), [CSS Variables Theme](https://cxjs.io/docs/concepts/css-variables-theme.md) |
| Tailwind setup | [Tailwind CSS](https://cxjs.io/docs/intro/tailwind-css.md) |
| how theming works (class names, state maps) | [Theming](https://cxjs.io/docs/concepts/theming.md), [Classic Themes](https://cxjs.io/docs/concepts/classic-themes.md) |
| smaller CSS bundles | [CSS Tree-Shaking](https://cxjs.io/docs/concepts/css-tree-shaking.md) |
