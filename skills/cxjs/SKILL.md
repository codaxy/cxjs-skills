---
name: cxjs
description: Use when writing, reviewing, refactoring or debugging CxJS code — any project that depends on the `cx` package. Covers typed models, store and bindings, controllers, widgets, forms, grids, charts, routing and theming.
---

# CxJS

**CxJS is not React.** It uses JSX and renders through React, but a CxJS tree is a declarative *widget configuration*, not a function that runs on every render. There are no hooks and no component-local state. All state lives in one store; widgets bind to paths in it, and the framework re-renders only what changed.

Most mistakes come from treating it like React: a hook has nowhere to live, a variable updated in a handler is read by nothing, and a value placed where a child widget is expected is not printed.

## Where to look

Work through these in order and stop as soon as you have the answer.

1. **The project itself.** A project skill, `AGENTS.md`, or the code around the change. Its conventions win over this skill; do not migrate an existing codebase to a different style unless asked.
2. **This skill and its references.** Conventions and pitfalls that the documentation does not spell out.
3. **The documentation** — concepts, widget usage, and working examples. Every page is available as Markdown at `https://cxjs.io/docs/<section>/<page>.md`; `https://cxjs.io/llms.txt` indexes all of them.
4. **The source** — only when you need **exact typing** or an **internal mechanism** the documentation does not cover. The `cx` package ships its TypeScript source, so it is always at hand and always matches the installed version:
   - the import path maps to a folder: `cx/widgets` → `node_modules/cx/src/widgets/`, likewise `cx/ui`, `cx/data`, `cx/util`, `cx/charts`, `cx/svg`;
   - a widget's props are the `XxxConfig` interface in its file — search for `export interface GridConfig`, `GridColumnConfig`, `TextFieldConfig`, …;
   - defaults are set on the prototype right after the class: `Grid.prototype.showHeader = true`, `Grid.prototype.recordName = "$record"`.

The documentation describes the latest release. When it disagrees with the installed source, the source is right for this project — check the `cx` version in `package.json`.

## Before writing code

- **Confirm the JSX setup.** `tsconfig.json` should have `"jsx": "react-jsx"` and `"jsxImportSource": "cx"`. With that runtime every JSX expression is a CxJS configuration and no wrapper is needed.
- **`<cx>` wrappers.** Older projects, and projects compiled with `swc-plugin-transform-cx-jsx` or `babel-plugin-transform-cx-jsx`, wrap CxJS JSX in `<cx>…</cx>`. If the project does, follow it everywhere — including JSX in expression positions such as a ternary arm, a column's `items`, or an `onResolve` result. Do not add `<cx>` to a project that does not use it.
- **Read the folder's `model.ts` and `Controller.ts`** before changing a view, and check the project's shared components before building a new one.

## Data

### Typed models

All store access goes through typed accessors created with `createModel<T>()` from `cx/data`. Never use string paths (`bind("user.name")`, `store.get("user.name")`) in new code — they are invisible to TypeScript and to rename refactors.

```ts
// model.ts
import { createModel } from "cx/data";

interface PageModel {
  user: { firstName: string; lastName: string };
  items: Item[];
  $item: Item;
}

export default createModel<PageModel>();
```

Pass an accessor directly to a widget prop for two-way binding: `<TextField value={m.user.firstName} />`. Use `bind(m.x, defaultValue)` only to supply a default. See [code organization](references/code-organization.md) for model files, record aliases and multiple models.

### Derived values

| Need | Use |
|---|---|
| a value as-is | the accessor: `value={m.name}` |
| a simple condition | a helper from `cx/ui`: `truthy`, `falsy`, `isEmpty`, `isNonEmpty`, `hasValue`, `equal`, `strictEqual`, `greaterThan`, … |
| a cheap derivation — comparison, ternary, string concatenation, arithmetic | `expr(a, b, (a, b) => …)` |
| an expensive derivation — array `filter`/`map`/`reduce`/sort, aggregation, lookups in collections | `computable(a, b, (a, b) => …)` — recomputes only when an input changes |
| text composed from values | `tpl(a, b, "{0} of {1}")`, with formats: `"{0:n;2}"`, `"{0:d|—}"` |
| one formatted value | `format(a, "currency;USD")`, or a column's `format` |

Never write `expr(x, (v) => !!v)` — that is `truthy(x)`.

### The store

Read and write through the store: `this.store` in a controller, `instance.store` (or the destructured `{ store }`) in an event handler.

| Call | Does |
|---|---|
| `store.get(m.x)` / `store.set(m.x, v)` | read / write |
| `store.init(m.x, v)` | write only if the value is `undefined` — use for seeding |
| `store.update(m.x, (x) => …)` | functional update |
| `store.toggle(m.x)` | flip a boolean — never `store.set(p, !store.get(p))` |
| `store.delete(m.x)` | remove |

The store is immutable. Never mutate an object or array you got from it — create a new one (`[...items, item]`, `{ ...user, name }`); a mutation triggers no update.

Store raw values and format them at the binding. Do not write `formattedDate` or `displayLabel` fields into the store.

## Rendering

- **Bound text goes in `text`**: `<div text={m.name} />`, `<span text={tpl(m.count, "{0} items")} />`. A binding placed as a child — `<div>{m.name}</div>` — is treated as a widget configuration and is not printed. Static literal text as a child is fine.
- **Conditions use `visible`** (alias `if`): `<div visible={truthy(m.error)} />`. Use `PureContainer` to toggle a group without a wrapper element, and `FirstVisibleChildLayout` to render only the first visible child — the idiomatic if/else chain.
- **CSS classes**: prefer `class`. It accepts a string or a structured object, `class={{ active: truthy(m.selected) }}`, where each entry is independently reactive. `className` behaves the same and combines with `class` — use it for the static part when a widget also has dynamic classes. See [styling](references/styling.md).

## Controllers

A controller holds a view's behaviour. Attach it with the `controller` prop — never as a child element.

```ts
// Controller.ts
import { Controller } from "cx/ui";
import m from "./model";

export default class extends Controller {
  onInit() {
    this.store.init(m.page, 1);
    this.addTrigger("load", [m.filter, m.page], (filter, page) => this.load(filter, page), true);
  }

  async load(filter: Filter, page: number) {
    // …
  }
}
```

- **Reactions** go through `this.addTrigger(name, [deps], callback, autoRun)`, not `store.subscribe` and not widget `onChange`. `autoRun: true` runs it once immediately — the usual way to load data.
- **Derived values written to the store**: `this.addComputable(m.total, [m.items], (items) => …)`. Prefer a `computable` at the binding unless several places read the result.
- **Clean up** timers, subscriptions and keyboard shortcuts in `onDestroy`.
- **Call controller methods through `getControllerByType`**, which is type-checked. Never use string handlers such as `onClick="onSave"`.

```tsx
import Controller from "./Controller";

<Button text="Save" onClick={(e, instance) => instance.getControllerByType(Controller).onSave()} />;
```

## TypeScript

**Leave callback parameters unannotated.** CxJS callbacks are typed by the prop they are passed to — `onClick`, `onResolve`, `onValidate`, `onQuery`, `onCreateFilter`, `onDrop`, `addTrigger` and `addComputable` callbacks, `expr` and `computable` functions. Write `onClick={(e, instance) => …}`, not `onClick={(e: any, instance: Instance) => …}`. Do not add `any`, and do not spend time searching for the parameter types — the inferred ones are correct.

Annotate only where TypeScript has no context: controller method parameters, standalone functions, and functional component props interfaces.

Do not cast bindings to silence type errors (`m.x as any`). A type error on a binding usually means the model is wrong — fix the model.

## Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Tooltips do not appear | tooltips are not enabled | call `enableTooltips()` from `cx/widgets` once at startup |
| `confirm` on a button shows the browser dialog | CxJS message boxes are not enabled | call `enableMsgBoxAlerts()` from `cx/widgets` once at startup |
| a prop passed to a functional component is always `undefined` inside | `visible`, `if`, `controller`, `layout`, `outerLayout`, `putInto`, `contentFor` and `vdomKey` are applied to the component's wrapper and removed from its props | do not redeclare or forward them — they already work at the call site |
| a window inside a functional component cannot be closed | the component was given a `visible` prop, which it strips, so the `Window` never gets a binding to write `false` into | name the prop differently (`shown`, `active`) and bind the window: `visible={shown}` |
| a widget default stops working after a prop is added | an explicit `undefined` in a config overwrites the prototype default | spread optional props conditionally: `...(step != null && { minTickStep: step })` |
| a chart does not appear | the `Svg` has no height — it measures itself and passes its size to the chart | give the `Svg` a CSS height, or `aspectRatio` with `autoHeight` |
| a Grid loses scroll position and selection on every change | it sits inside a `ContentResolver`, which rebuilds its subtree whenever `params` change | bind data straight into the Grid; keep `ContentResolver` for genuinely structural changes |
| `m.$record.x` is not typed, or a nested repeater shows the wrong record | the record alias is not declared in the model, or two collections share `$record` | declare the alias in the model and pass `recordAlias={m.$item}` |

## Legacy syntax

Recognise these in existing code; never write them in new code:

- attribute suffixes: `value-bind="x"`, `value:bind="x"`, `text-tpl="…"`, `visible-expr="…"`;
- string paths and string templates: `bind("x")`, `{ bind: "x" }`, `tpl("{x}")`, `{ expr: "…" }` — the object form itself is fine with an accessor, e.g. `{ bind: m.x, debounce: 300 }`;
- `createAccessorModelProxy` — the old name of `createModel`;
- `<cx>` wrappers in a project that uses the `jsxImportSource: "cx"` runtime.

## References

Read the file that matches the task.

| File | Covers |
|---|---|
| [references/code-organization.md](references/code-organization.md) | folders, file names, exports, model and controller files, inline props, plain JSX vs functional components, prop types, widget classes |
| [references/app-structure.md](references/app-structure.md) | entry point, root model, page data with `SandboxedRoute`, routes, navigation, lazy loading |
| [references/grids.md](references/grids.md) | `Grid`: columns, cell rendering, grouping and aggregates, row actions, server-side sorting and paging, document tables, large data sets |
| [references/forms.md](references/forms.md) | form layout (`LabelsTopLayout`), binding, validation and submitting, search boxes, lookups |
| [references/styling.md](references/styling.md) | who styles what, `class`/`className`, `mod`, `cx-theme-variables`, Tailwind setup, classic themes |
| [references/charts.md](references/charts.md) | `Svg` and `Chart`, axes, anchors/offset/margin, series, colours, legends |
| [references/overlays.md](references/overlays.md) | windows (declared, opened from code, hot promise windows), dropdowns, confirmations, toasts, tooltips |
