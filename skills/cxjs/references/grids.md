# Grids

`Grid` (from `cx/widgets`) covers sorting, selection, grouping with aggregates, cell editing, trees, buffering and drag & drop. The [Grid documentation](https://cxjs.io/docs/tables/grid.md) lists every option; this file covers the conventions and the parts that are easy to get wrong.

## A typical grid

```ts
// model.ts
interface PageModel {
  orders: Order[];
  selectedId: string;
  $order: Order;
  $group: { region: string; count: number };
}
```

```tsx
<Grid
  records={m.orders}
  recordAlias={m.$order}
  keyField="id"
  emptyText="No orders found."
  selection={{ type: KeySelection, bind: m.selectedId, keyField: "id" }}
  columns={[
    { header: "Customer", field: "customer", sortable: true },
    { header: "Date", field: "date", format: "d", sortable: true },
    { header: "Total", field: "total", format: "currency;;2", align: "right", sortable: true },
    { header: "Status", field: "status", children: <StatusTag status={m.$order.status} /> },
  ]}
/>
```

- Declare the record alias in the model and pass it with `recordAlias={m.$order}`. Use `$record` only when the view has a single collection.
- Set `keyField` — it lets the grid track rows across sorting and data changes.
- Set `emptyText`.
- Columns go inline while they are short. Once the list grows past about 10 lines, move it to `columns.tsx` next to the view (see [code organization](code-organization.md)).

## Columns

`field` is a plain string naming a record property. Choose how a cell renders from the simplest option that works:

| The cell shows | Use |
|---|---|
| a property as-is | `field` |
| a property, formatted or with a null fallback | `format`: `"n;2"`, `"d"`, `"currency;USD"`, `"s\|N/A"`, `"n;0\|0"` |
| a value derived in code | `value: expr(m.$order.qty, m.$order.price, (q, p) => q * p)` |
| markup — a link, a tag, several elements, buttons | `children: <…/>` |

- Prefer `format` over a `value` that only formats.
- Custom cell content goes in `children`, which renders any widgets for the current record.
- A column with a computed `value` sorts by that value. Add `sortField` to sort by a raw record field instead.
- `columns` itself is not bindable. To change columns at runtime, use `columnParams` + `onGetColumns`, which recalculates the columns whenever the params change ([Dynamic Columns](https://cxjs.io/docs/tables/dynamic-columns.md)).

## Grouping and aggregates

```tsx
<Grid
  records={m.orders}
  recordAlias={m.$order}
  grouping={[
    { showFooter: true }, // grand total
    { key: { region: m.$order.region }, showCaption: true },
  ]}
  columns={[
    {
      header: "Customer",
      field: "customer",
      aggregate: "count",
      aggregateAlias: "count",
      caption: tpl(m.$group.region, "{0}"),
      footer: tpl(m.$group.count, "{0} orders"),
    },
    { header: "Total", field: "total", format: "currency;;2", aggregate: "sum", align: "right" },
  ]}
/>
```

- Group values and aggregates are exposed under `$group` — declare it in the model to bind to it with a typed accessor.
- Aggregates: `sum`, `count`, `avg`, `distinct`, `min`, `max`. An aggregate is published under the column's `field` unless `aggregateAlias` names it.
- Write `footer` and `caption` with typed accessors — `tpl(m.$group.count, "{0} orders")` — not string templates like `{ tpl: "{$group.count}" }`.
- To change grouping at runtime, use `groupingParams` + `onGetGrouping` ([Dynamic Grouping](https://cxjs.io/docs/tables/dynamic-grouping.md)).

## Actions in rows

```tsx
{
  header: "Actions",
  align: "center",
  children: (
    <Button
      mod="hollow"
      icon="trash"
      onMouseDown={stopPropagation}
      onClick={(e, instance) =>
        instance.getControllerByType(Controller).removeOrder(instance.store.get(m.$order.id))
      }
    />
  ),
}
```

- Inside a row, `instance.store` sees the row's record: `instance.store.get(m.$order)`.
- In a selectable grid, add `onMouseDown={stopPropagation}` (from `cx/util`) to buttons so that clicking them does not also select the row.

## Server-side sorting and paging

The grid writes the clicked column into its sort bindings; with `remoteSort` it does not sort locally. A controller trigger reloads when the sort, the page or the filter changes.

```tsx
<Grid
  records={m.orders}
  recordAlias={m.$order}
  keyField="id"
  remoteSort
  sortField={m.sortField}
  sortDirection={m.sortDirection}
  lockColumnWidths
  columns={[ /* … */ ]}
/>
<Pagination page={m.page} pageCount={m.pageCount} />
```

```ts
export default class extends Controller {
  onInit() {
    this.store.init(m.page, 1);
    this.addTrigger(
      "load",
      [m.filter, m.page, m.sortField, m.sortDirection],
      (filter, page, sortField, sortDirection) => this.load(filter, page, sortField, sortDirection),
      true,
    );
  }

  async load(filter: Filter, page: number, sortField: string, sortDirection: SortDirection) {
    let result = await listOrders({ filter, page, sortField, sortDirection });
    this.store.set(m.orders, result.records);
    this.store.set(m.pageCount, result.pageCount);
  }
}
```

- `lockColumnWidths` keeps column widths stable as pages change.
- Reset `page` to 1 when the filter changes.
- Guard against a slow earlier response overwriting a newer one when requests can overlap.
- For an endless list, use `infinite` with `onFetchRecords` instead ([Infinite Scrolling](https://cxjs.io/docs/tables/infinite-scrolling.md)).

## Grids as document tables

When a grid is content inside a document — a report, a detail page, a card — it should look like a plain table rather than an interactive data grid. Use `headerMode="plain"`: the header gets no background and no cell borders.

```tsx
<Grid
  records={m.order.lines}
  recordAlias={m.$line}
  headerMode="plain"
  columns={[
    { header: "Item", field: "name", sortable: true },
    { header: "Qty", field: "qty", format: "n;0", align: "right" },
    { header: "Amount", field: "amount", format: "currency;;2", align: "right" },
  ]}
/>
```

If `headerMode` is not set, the grid chooses on its own: `plain` when nothing is interactive, `default` as soon as the grid is `scrollable` or any column is `sortable` or resizable. Set `headerMode="plain"` explicitly when a document table has sortable columns but should still look plain. Leave it out of scrollable data grids, and avoid selection in document tables.

## Large data sets

- `buffered` renders only the visible rows and turns on `scrollable`. The grid needs a fixed height: `style="height: 400px"` or a flex layout that gives it one.
- `cached` skips re-rendering rows whose record did not change.
- `mod="fixed-layout"` switches the table to `table-layout: fixed` — faster with many rows; give the columns explicit widths.
- Set `keyField`.

## Pitfalls

- **Never place a Grid inside a `ContentResolver`.** It rebuilds its subtree whenever its `params` change, which resets scroll position and selection. Bind data straight into the grid; it updates incrementally.
- **Do not mutate records.** Replace the array (`store.update(m.orders, (orders) => …)`) — see the immutable array helpers in `cx/data` (`append`, `updateArray`, `insertElement`, `moveElement`).
- **A scrollable grid needs a height.** Without one it grows with its content and never scrolls.

## Documentation

| Task | Page |
|---|---|
| all options, columns, headers | [Grid](https://cxjs.io/docs/tables/grid.md) |
| selection | [Selections](https://cxjs.io/docs/concepts/selections.md), [Multiple Selection](https://cxjs.io/docs/tables/multiple-selection.md) |
| filtering and search | [Searching and Filtering](https://cxjs.io/docs/tables/searching-and-filtering.md) |
| paging | [Pagination](https://cxjs.io/docs/tables/pagination.md), [Infinite Scrolling](https://cxjs.io/docs/tables/infinite-scrolling.md), [Buffering](https://cxjs.io/docs/tables/buffering.md) |
| grouping | [Grouping](https://cxjs.io/docs/tables/grouping.md), [Dynamic Grouping](https://cxjs.io/docs/tables/dynamic-grouping.md) |
| editing | [Cell Editing](https://cxjs.io/docs/tables/cell-editing.md), [Inline Edit](https://cxjs.io/docs/tables/inline-edit.md), [Row Editing](https://cxjs.io/docs/tables/row-editing.md), [Form Edit](https://cxjs.io/docs/tables/form-edit.md) |
| trees | [Tree Grid](https://cxjs.io/docs/tables/tree-grid.md), [Tree Operations](https://cxjs.io/docs/tables/tree-operations.md) |
| columns | [Complex Headers](https://cxjs.io/docs/tables/complex-headers.md), [Fixed Columns](https://cxjs.io/docs/tables/fixed-columns.md), [Column Resizing](https://cxjs.io/docs/tables/column-resizing.md), [Column Reordering](https://cxjs.io/docs/tables/column-reordering.md), [Dynamic Columns](https://cxjs.io/docs/tables/dynamic-columns.md), [Header Menu](https://cxjs.io/docs/tables/header-menu.md) |
| drag & drop | [Row Drag and Drop](https://cxjs.io/docs/tables/row-drag-and-drop.md), [Tree Drag and Drop](https://cxjs.io/docs/tables/tree-drag-and-drop.md) |
| row details | [Row Expanding](https://cxjs.io/docs/tables/row-expanding.md) |
