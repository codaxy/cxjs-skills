# Code organization

How to split CxJS code into folders and files, what to name, what to export, and when to create a component.

The guiding idea: **naming things is hard and remembering names is harder.** CxJS code relies on *proximity* — where a file sits tells you what it does — and keeps the set of exported names small. A reader should be able to understand a view top to bottom without jumping around.

## Folders

A folder groups everything that belongs to one feature, typically a route:

```
routes/orders/
├── index.tsx       # the view (default export)
├── Controller.ts   # behaviour (default export)
├── model.ts        # the store shape (default export)
├── columns.tsx     # optional: a large extracted prop
└── Summary.tsx     # optional: a part of a large view
```

The folder decides what leaves it. Code outside the folder imports its entry point, never its internals; helpers, parts and styles inside stay private.

## Naming

A file is named after what it exports:

| The file exports | Casing | Examples |
|---|---|---|
| a class | PascalCase | `Controller.ts`, `DateRangePicker.tsx` |
| something used as a JSX tag (a functional component, a plain JSX part) | PascalCase | `StatusTag.tsx`, `Summary.tsx` |
| a value (a model, a utility, a config) | camelCase | `model.ts`, `editorModel.ts`, `columns.tsx`, `formatDuration.ts` |
| a folder's entry point | `index.tsx` | `routes/orders/index.tsx` |

Folders are kebab-case (`routes/order-details/`). Record aliases are `$` + camelCase (`$order`, `$orderLine`).

## Exports

Choose the export by how many places use the thing:

| Used | Export | Example |
|---|---|---|
| in exactly one place (a route, a page part, a controller, a model) | `export default`, unnamed — the importer names it | `import Orders from "./orders"` |
| in many places (a shared component, a utility) | named export | `export const StatusTag = …` |

A default export is a deliberate signal that the thing has one home. There is no `OrdersRoute` name for the rest of the codebase to discover and reuse in the wrong place; the one place that uses it names it.

Export only what another module actually imports. Do not export for convenience.

## Files

- Small pieces stay in the file that uses them.
- A larger piece may move to its own file next to its user — a judgement call, not a rule.
- Anything used by more than one module gets its own file.

## Keep props inline

Keep props inline in the JSX — computed values, config objects, option lists, handlers — so the view reads in one pass. Extract a prop only when its value is **more than 10 lines**; the typical case is a grid's `columns`.

```tsx
// ✗ small values hoisted — the reader has to jump around
const fullName = expr(m.firstName, m.lastName, (f, l) => `${f} ${l}`);
const selection = { type: KeySelection, bind: m.selectedId, keyField: "id" };
<div text={fullName} />
<Grid records={m.orders} selection={selection} columns={columns} />

// ✓ inline
<div text={expr(m.firstName, m.lastName, (f, l) => `${f} ${l}`)} />
<Grid
  records={m.orders}
  selection={{ type: KeySelection, bind: m.selectedId, keyField: "id" }}
  columns={columns}
/>
```

A long event handler does not become a local function — it becomes a controller method.

## The model

Each folder usually has one `model.ts`, shared by its controller and its views. It default-exports the model proxy:

```ts
// model.ts
import { createModel } from "cx/data";

interface PageModel {
  orders: Order[];
  selectedId: string;
  selectedOrder: Order;
  $order: Order; // record alias for the orders grid
  $line: OrderLine; // record alias for the lines repeater
}

export default createModel<PageModel>();
```

- **Never export a model as a named export.** A view often uses several models — the page model, a global app model (user, permissions), a model for a `Restate` — and each importer names them as it sees fit.
- Export the interface only if another file uses it.
- Declare record aliases in the model. `$record` is fine when there is one collection; when collections compete, give each its own alias and pass it explicitly: `recordAlias={m.$order}`.

```tsx
import m from "./model";
import app from "../../model"; // global: user, permissions, …
import editor from "./editorModel"; // the private store inside Restate

export default (
  <div controller={Controller}>
    <div text={app.user.name} />
    <Grid records={m.orders} recordAlias={m.$order} columns={columns} />
    <Restate data={{ order: m.selectedOrder }}>
      <TextField value={editor.order.name} />
    </Restate>
  </div>
);
```

Inside a `Restate` bind only through the model that describes its private store.

## The controller

`Controller.ts` default-exports an unnamed class — its location says what it controls:

```ts
// Controller.ts
import { Controller } from "cx/ui";
import m from "./model";

export default class extends Controller {
  onInit() {
    this.store.init(m.selectedId, null);
  }

  onSave() {
    // …
  }
}
```

Call its methods through `getControllerByType`, which is type-checked:

```tsx
import Controller from "./Controller";

<Button
  text="Save"
  onClick={(e, instance) => instance.getControllerByType(Controller).onSave()}
/>;
```

- **Never use string handlers** (`onClick="onSave"`). They are not type-checked and break silently when a method is renamed.
- Call `getControllerByType` on the instance; do not destructure it — it relies on `this`.
- Trivial UI-only logic can stay inline: `onClick={(e, { store }) => store.toggle(m.expanded)}`.

## Plain JSX vs functional components

Use `createFunctionalComponent` **only when the component has its own props.** A view without props is plain CxJS JSX bound to the model:

```tsx
export default <div text={m.title} />;
```

Split a large view into several files of plain JSX that import the shared model. The parts need no props and no functional component:

```tsx
// Summary.tsx
import { tpl } from "cx/ui";
import m from "./model";

export default <div class="summary" text={tpl(m.orders.length, "{0} orders")} />;
```

```tsx
// index.tsx
import Summary from "./Summary";

export default (
  <div controller={Controller}>
    <Summary />
    …
  </div>
);
```

`<Summary />` works because the CxJS JSX runtime passes an existing widget configuration through unchanged — any attributes on it are ignored.

## Functional component props

Type props by what the component does with them:

| The prop is | Type it as |
|---|---|
| data the component reads, writes or derives from | `AccessorChain<T>` |
| passed straight through to an inner widget (`class`, `className`, `style`, `text`, `disabled`, …) | the matching CxJS prop type: `ClassProp`, `StyleProp`, `StringProp`, `BooleanProp`, … |
| static configuration, fixed when the component is created | a plain TypeScript type |

```tsx
import { AccessorChain, ClassProp, createFunctionalComponent, expr } from "cx/ui";

interface StatusTagProps {
  status: AccessorChain<Status>; // used inside
  class?: ClassProp; // forwarded unchanged
}

export const StatusTag = createFunctionalComponent(({ status, class: className }: StatusTagProps) => (
  <span class={className} text={expr(status, (s) => labels[s])} />
));
```

Prefer `AccessorChain<T>` wherever possible — it is fully typed and works with `expr`, `computable` and `tpl`. The trade-off: callers cannot pass a literal or a computed value to it, so widen to `Prop<T>` only when they genuinely need to.

## Widget classes

CxJS is about composition and data flow, not low-level rendering. Almost everything is plain JSX or a functional component that combines existing widgets.

Write a `Widget` class only when you need React capabilities:

- the component has **internal state** of its own (not store data);
- it **interacts directly with the DOM** — measuring, focus, scrolling, event listeners;
- it **wraps a third-party library** (an editor, a map, a canvas) into CxJS.

A widget file uses React JSX (`/** @jsxImportSource react */`), declares a `XxxConfig` interface with bindable prop types, uses `declare` for class fields, registers bindable props in `declareData`, renders in `render(context, instance, key)`, and sets defaults on the prototype. For the details:

- the docs: [Custom Components](https://cxjs.io/docs/intro/custom-components.md) and [Authoring Widgets](https://cxjs.io/docs/intro/ts-migration-guide.md) in the TypeScript migration guide;
- a small real example in the installed package: `node_modules/cx/src/widgets/ProgressBar.tsx`;
- the base class you extend (`Widget`, `HtmlElement`, `Field`, `ContainerBase`, `PureContainerBase`) in `node_modules/cx/src/ui/` and `node_modules/cx/src/widgets/`.
