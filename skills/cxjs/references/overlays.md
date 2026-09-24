# Overlays

Windows, confirmations and toasts, all from `cx/widgets`. The [Window](https://cxjs.io/docs/layout/window.md), [MsgBox](https://cxjs.io/docs/layout/msgbox.md) and [Toast](https://cxjs.io/docs/layout/toast.md) pages list every option.

## Windows

Both ways of showing a window are common; pick the one that fits.

### Declared in the view

The window sits in the view and its `visible` is bound to the store:

```tsx
<Window title="Edit order" visible={m.editor.visible} center modal draggable closeOnEscape>
  <LabelsTopLayout vertical mod="top">
    <TextField label="Customer" value={m.order.customer} required />
  </LabelsTopLayout>
  <div putInto="footer">
    <Button text="Save" mod="primary" onClick={(e, instance) => instance.getControllerByType(Controller).onSave()} />
    <Button text="Cancel" dismiss />
  </div>
</Window>
```

- The window closes itself — close button, Escape, `dismiss` on a button — by writing `false` into its `visible` binding. **`visible` must therefore be a binding**, not a computed value.
- `putInto="footer"` places buttons in the window footer. `dismiss` on a `Button` closes the enclosing window.

### Closing a window from code

A window passes its `dismiss` function through `parentOptions` — this works for declared windows and for windows opened from code alike. A controller on the window closes it once its work is done:

```tsx
<Window title="Edit order" visible={m.editor.visible} center modal controller={EditorController}>
  …
  <Button text="Save" mod="primary" onClick={(e, instance) => instance.getControllerByType(EditorController).onSave()} />
</Window>
```

```ts
// EditorController
async onSave() {
  await saveOrder(this.store.get(m.order));
  this.instance.parentOptions.dismiss();
}
```

- Before `cx` 26.9.3 a controller attached to the `Window` itself did not receive `dismiss`; in projects on an older version, attach the controller to an element inside the window instead.
- An event handler inside the window can do the same: `onClick={(e, instance) => instance.parentOptions.dismiss()}`.

**Wrapping a window in a functional component: do not call the prop `visible`.** A functional component strips `visible` and applies it to its own wrapper, so the window inside never receives the binding and cannot close itself. Use another name — `shown`, `active` — and pass it on:

```tsx
interface OrderEditorProps {
  shown: AccessorChain<boolean>;
}

export const OrderEditor = createFunctionalComponent(({ shown }: OrderEditorProps) => (
  <Window title="Edit order" visible={shown} center modal>
    …
  </Window>
));

<OrderEditor shown={m.editor.visible} />;
```

### Opened from code

Create the window and open it when needed. `open` takes a store or an instance and returns a function that closes the window:

```tsx
onClick={(e, instance) => {
  Window.create(
    <Window title="Order details" center modal>
      <div text={m.$order.customer} />
    </Window>,
  ).open(instance, { initiatingEvent: e });
}}
```

- **Pass the instance** to give the window the data of the place it was opened from — a grid row's record, for example — and access to its controllers through `getControllerByType`.
- Pass `new Store({ data })` instead for a window with its own isolated data.
- Pass `{ initiatingEvent: e }` so the overlay container is configured from where it was opened.
- To get a result back, use a hot promise window (below).

### Hot promise windows

For a dialog that returns a result — "edit this record and give it back" — use `createHotPromiseWindowFactoryWithProps` (or `createHotPromiseWindowFactory` without props). It returns a function that opens the window and resolves a Promise when it closes. It also keeps the window working with hot module replacement: edits to its content show up in an open window instead of requiring it to be closed and reopened.

```tsx
// openOrderEditor.tsx
import { Button, createHotPromiseWindowFactoryWithProps, LabelsTopLayout, TextField, Window } from "cx/widgets";
import m from "./editorModel"; // { order: Order }

export const openOrderEditor = createHotPromiseWindowFactoryWithProps(
  module,
  (title: string) => (resolve: (order: Order | false) => void) => {
    let result: Order | false = false;
    return Window.create(
      <Window title={title} center modal onDestroy={() => resolve(result)}>
        <LabelsTopLayout vertical mod="top">
          <TextField label="Customer" value={m.order.customer} required />
        </LabelsTopLayout>
        <div putInto="footer">
          <Button text="Save" mod="primary" dismiss onClick={(e, { store }) => (result = store.get(m.order))} />
          <Button text="Cancel" dismiss />
        </div>
      </Window>,
    );
  },
);
```

```ts
// in a controller
const order = await openOrderEditor("Edit order", { store: new Store({ data: { order: selected } }) });
if (order) this.store.update(m.orders, (orders) => orders.map((o) => (o.id == order.id ? order : o)));
```

- Resolve in `onDestroy`, so closing with Escape or the close button also settles the Promise.
- The second argument chooses the window's data: `{ store: new Store({ data }) }` for an isolated store, or `{ parent: instance }` to share the caller's store.
- The first argument is the hot module. Pass `module` with webpack; with Vite pass `{ hot: import.meta.hot }` and also call `if (import.meta.hot) import.meta.hot.accept();` in that file — Vite only treats a module as self-accepting when it sees that call.

## Dropdowns

`Dropdown` positions content next to another element — by default the element rendered just before it — and picks the placement with the most room.

```tsx
<Button text="Options" onClick={(e, { store }) => store.toggle(m.showOptions)} />
<Dropdown visible={m.showOptions} arrow offset={4} placementOrder="down-right up-right">
  …
</Dropdown>
```

- `placementOrder` lists the allowed placements in order of preference; `offset` sets the distance; `arrow` points at the anchor; `matchWidth` matches the anchor's width.
- `onResolveRelatedElement={(el) => el.parentElement}` anchors it to a different element.
- For search suggestions, bind a field's focus: `<TextField value={m.query} focused={m.showSuggestions} trackFocus />` followed by `<Dropdown visible={m.showSuggestions} matchWidth>`.

For a dropdown that opens from a menu or toolbar, put the content in a `MenuItem` with `putInto="dropdown"`:

```tsx
<Menu horizontal>
  <MenuItem clickToOpen dropdownOptions={{ dismissOnFocusOut: true, constrain: true }}>
    Filters
    <div putInto="dropdown">…filter fields…</div>
  </MenuItem>
</Menu>
```

Fields that open their own dropdowns — `LookupField`, `DateField` — inside such a panel need `dismissOnFocusOut` in `dropdownOptions`. It makes the panel a focusable overlay container, so the nested dropdowns stay inside it and do not close the panel when focus moves into them. `constrain` keeps the panel within the viewport.

## Confirmations

Prefer the `confirm` prop on a button — the handler runs only if the user confirms:

```tsx
<Button text="Delete" mod="danger" confirm="Delete this order?" onClick={(e, instance) => instance.getControllerByType(Controller).onDelete()} />
```

Inside more complex logic — a submit that validates first, then asks — use `MsgBox.yesNo`, which resolves to `"yes"` or `"no"`:

```ts
async onSubmit() {
  if (this.store.get(m.invalid)) {
    this.store.set(m.visited, true);
    return;
  }
  if ((await MsgBox.yesNo("Submit the order for processing?")) !== "yes") return;
  // submit
}
```

Both need `enableMsgBoxAlerts()` called once at startup; without it `confirm` falls back to the browser's dialog.

## Toasts

```ts
Toast.create({ message: "Order saved.", mod: "success", timeout: 3000 }).open(instance.store);
```

Toasts appear at the top by default; `placement` changes that.

## Tooltips

The `tooltip` prop takes a string or a configuration. Tooltips require `enableTooltips()` once at startup. Use `tooltip`, not the HTML `title` attribute.

## Documentation

| Task | Page |
|---|---|
| windows | [Window](https://cxjs.io/docs/layout/window.md), [Overlay](https://cxjs.io/docs/layout/overlay.md) |
| confirmations and alerts | [MsgBox](https://cxjs.io/docs/layout/msgbox.md) |
| notifications | [Toast](https://cxjs.io/docs/layout/toast.md) |
| tooltips | [Tooltip](https://cxjs.io/docs/layout/tooltip.md) |
| menus | [Menu](https://cxjs.io/docs/layout/menu.md), [ContextMenu](https://cxjs.io/docs/layout/context-menu.md), [Dropdown](https://cxjs.io/docs/layout/dropdown.md) |
