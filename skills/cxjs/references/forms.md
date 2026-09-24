# Forms

Form fields (`TextField`, `NumberField`, `DateField`, `LookupField`, `Checkbox`, `Switch`, …) come from `cx/widgets`, bind with `value`, and validate themselves. This file covers layout, binding, validation and submitting; the [forms documentation](https://cxjs.io/docs/forms/text-field.md) has every field and option.

## Layout

Always lay out fields with a CxJS label layout — not with custom markup or utility classes. **`LabelsTopLayout` is the default choice**:

```tsx
<LabelsTopLayout columns={2} mod={["stretch", "fixed"]}>
  <TextField label="First name" value={m.order.firstName} required />
  <TextField label="Last name" value={m.order.lastName} required />
  <LabelsTopLayoutCell colSpan={2}>
    <TextField label="Address" value={m.order.address} style="width: 100%" />
  </LabelsTopLayoutCell>
  <DateField label="Delivery date" value={m.order.deliveryDate} />
  <NumberField label="Quantity" value={m.order.qty} format="n;0" />
</LabelsTopLayout>
```

- `columns` arranges fields in a grid; `vertical` puts each field on its own row.
- `LabelsTopLayoutCell` spans a field across columns or rows (`colSpan`, `rowSpan`).
- Mods — combine them with an array, e.g. `mod={["stretch", "fixed"]}`:

  | Mod | Effect | Use |
  |---|---|---|
  | `stretch` | the layout table takes the full width (`width: 100%`) | forms that fill their container |
  | `fixed` | `table-layout: fixed` — columns get equal widths | multi-column forms that should align, usually together with `stretch` |
  | `top` | removes the top padding of labels in the first row | forms inside a `Window` or a card, where that padding adds an unwanted gap |

  The mods size the table, not the inputs — a field keeps the theme's default width. For fields that fill their columns, set `style="width: 100%"` on them.
- `LabelsLeftLayout` places labels to the left, for short labels.
- To show or hide a group of fields without breaking the layout, wrap them in `<PureContainer visible={…} layout={UseParentLayout}>`.
- `LabeledContainer` puts several widgets under one label.

## Binding

Fields usually bind **directly to the record** being edited — especially inside windows. Use a separate draft copy only when the use case needs it, for example when the user must be able to cancel changes to data that is displayed elsewhere.

```tsx
<TextField label="Name" value={m.order.name} />
<LookupField label="Customer" value={m.order.customerId} text={m.order.customerName} onQuery={queryCustomers} />
```

When a field writes its value differs by widget — check `reactOn` if timing matters:

| Field | Writes to the store on |
|---|---|
| `TextField` | every keystroke |
| `NumberField`, `DateTimeField`, `MonthField` | Enter or blur |
| `TextArea` | blur |

A `TextField` that drives an expensive reaction — a search box that reloads a grid — should not write on every keystroke. Either debounce or throttle the binding:

```tsx
<TextField value={{ bind: m.filter.query, debounce: 300 }} placeholder="Search…" icon="search" />
```

- `debounce: ms` writes once the user pauses typing; `throttle: ms` writes at most once per interval while typing continues.
- Both need the object form `{ bind: accessor, … }` — a plain accessor (`value={m.filter.query}`) cannot carry them.
- Or set `reactOn="enter blur"` to write only when the user presses Enter or leaves the field.

## Validation

Wrap the form in a `ValidationGroup`. It tracks the validity of every field inside and exposes it through `valid` or `invalid`.

```tsx
<ValidationGroup invalid={m.invalid} visited={m.visited}>
  <LabelsTopLayout vertical>
    <TextField label="Email" value={m.order.email} required validationRegExp={/.+@.+\..+/} validationErrorText="Enter a valid email." />
    <TextField label="Password" value={m.order.password} inputType="password" required minLength={8} />
  </LabelsTopLayout>
</ValidationGroup>
<Button text="Save" mod="primary" onClick={(e, instance) => instance.getControllerByType(Controller).onSave()} />
```

```ts
onSave() {
  if (this.store.get(m.invalid)) {
    this.store.set(m.visited, true); // show which fields are invalid
    return;
  }
  // save
}
```

- **On submit, if the form is invalid, set `visited` on the group.** Fields only show their errors once visited; setting it on the group shows the user exactly which fields need attention.
- Field rules: `required`, `minLength`/`maxLength`, `validationRegExp`, `minValue`/`maxValue`, and `onValidate` for custom logic — return an error message, or `false` when valid. `onValidate` may return a Promise for server-side checks.
- Messages: `requiredText` for required, `validationErrorText` for `validationRegExp`, `minLengthValidationErrorText` for `minLength`.
- Cross-field rules go in a `Validator` inside the group: `<Validator value={{ a: m.password, b: m.confirm }} onValidate={({ a, b }) => a != b && "Passwords do not match."} />`.
- `validationMode` chooses where errors appear: `tooltip` (default — requires `enableTooltips()`), `help` or `help-block`.
- The group also propagates `disabled`, `readOnly`, `viewMode` and `asterisk` to every field inside.

## Lookups

```tsx
// static options — option objects are { id, text } unless optionIdField/optionTextField say otherwise
<LookupField label="Status" value={m.order.status} options={statusOptions} />

// remote options — keep value and text together so the label shows without a refetch
<LookupField label="Customer" value={m.order.customerId} text={m.order.customerName} onQuery={queryCustomers} />

// multiple selection — records holds the selected option objects, values holds their ids
<LookupField label="Tags" records={m.order.tags} options={tagOptions} multiple />
```

- `onQuery(query)` returns options or a Promise. Add `fetchAll` (and `cacheAll`) to fetch once and filter locally.
- For long remote lists use `infinite` with `onQueryPage({ query, page, pageSize })`.
- Single selection binds `value`/`text`; multiple selection binds `values` and/or `records`. Do not mix them.

## Documentation

| Task | Page |
|---|---|
| validation | [Validation](https://cxjs.io/docs/forms/validation.md), [ValidationGroup](https://cxjs.io/docs/forms/validation-group.md), [Validator](https://cxjs.io/docs/forms/validator.md) |
| layout | [LabelsTopLayout](https://cxjs.io/docs/layout/labels-top-layout.md), [LabelsLeftLayout](https://cxjs.io/docs/layout/labels-left-layout.md), [LabeledContainer](https://cxjs.io/docs/forms/labeled-container.md) |
| fields | [TextField](https://cxjs.io/docs/forms/text-field.md), [NumberField](https://cxjs.io/docs/forms/number-field.md), [DateField](https://cxjs.io/docs/forms/date-field.md), [Checkbox](https://cxjs.io/docs/forms/checkbox.md), [Radio](https://cxjs.io/docs/forms/radio.md), [Switch](https://cxjs.io/docs/forms/switch.md), [TextArea](https://cxjs.io/docs/forms/text-area.md) |
| lookups | [LookupField](https://cxjs.io/docs/forms/lookup-field.md), [Custom Lookup Bindings](https://cxjs.io/docs/forms/custom-lookup-bindings.md), [Infinite Lookup List](https://cxjs.io/docs/forms/infinite-lookup-list.md) |
