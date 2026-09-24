# Charts

CxJS charts are assembled from low-level parts rather than picked from a list of chart types: an `Svg` container, a `Chart` with axes, and graphs, markers and shapes inside it. `Svg` and the shapes come from `cx/svg`, everything else from `cx/charts`. The [charts documentation](https://cxjs.io/docs/charts/overview.md) covers every part.

## A typical chart

```tsx
<Legend.Scope>
  <Svg style="width: 100%; height: 300px">
    <Chart
      margin="20 20 40 50"
      axes={{
        x: { type: CategoryAxis },
        y: { type: NumericAxis, vertical: true },
      }}
    >
      <Gridlines />
      <ColumnGraph name="Turnover" data={m.months} xField="month" yField="turnover" colorIndex={4} size={0.3} offset={-0.15} />
      <ColumnGraph name="Expenses" data={m.months} xField="month" yField="expenses" colorIndex={0} size={0.3} offset={0.15} />
    </Chart>
  </Svg>
  <Legend />
</Legend.Scope>
```

- **`Svg` needs a height** — it measures itself and passes its size down. Give it a CSS height, or `aspectRatio` with `autoHeight`. Use `Svg`, not a native `<svg>`: only `Svg` provides the bounds its children lay themselves out in.
- **Axes** are keyed by name. Write each as a config object, `y: { type: NumericAxis, vertical: true }`, or as JSX, `y: <NumericAxis vertical />` — both are common. The vertical axis needs `vertical`.
- **Room for axis labels** comes from `margin` or `offset` on the `Chart` — both are common (see below). Without it the labels are clipped.
- `Gridlines` draws the grid; add `<Rectangle fill="white" />` as the first child for a plot background.

## Anchors, offset and margin

Every shape inside an `Svg` — `Chart`, `Rectangle`, `Text`, `Line`, … — is a *bounded object*: it computes its box from its parent's box. Three properties, each written as `"top right bottom left"` without units, control it. They are applied in this order:

1. **`anchors`** — where each edge sits, as a fraction of the parent box. `0` is the parent's top/left edge, `1` its bottom/right edge. The top and bottom values are fractions of the height; the right and left values fractions of the width.
2. **`offset`** — pixels added to each edge. Positive values move an edge right or down, negative values left or up. So `offset="10 -10 -10 10"` moves every edge 10px inward.
3. **`margin`** — like CSS margin: positive values shrink the box on every side. `margin="10 10 10 10"` (or `margin={10}`) gives the same box as the offset above.

`padding` works like `margin` but shrinks the box passed to the children instead of the element's own box.

| Want | Write |
|---|---|
| fill the parent — the default for `Chart`, `Rectangle` and `Line` | `anchors="0 1 1 0"` |
| fill the parent with a 10px inset | `margin={10}`, or `offset="10 -10 -10 10"` |
| the left half | `anchors="0 0.5 1 0"` |
| the bottom 40px strip | `anchors="1 1 1 0" offset="-40 0 0 0"` |
| a 60×40 box centred in the parent | `anchors="0.5 0.5 0.5 0.5" offset="-20 30 20 -30"` |
| a 20×20 box in the top-right corner | `anchors="0 1 0 1" offset="0 0 20 -20"` |
| a point at the centre — the default for `Text` | `anchors="0.5 0.5 0.5 0.5"` |

Two things to keep in mind:

- **All four anchors equal** collapses the box to a point; `offset` then grows it back into a box of a fixed pixel size around that point. This is how fixed-size elements are positioned relative to a proportional spot.
- **`offset` is always top-to-bottom, left-to-right**, so pixels that move the right or bottom edge *inward* are negative — the opposite of `margin`.

## Series

Two ways to plot data:

- **Graphs** take the whole array and field names: `LineGraph`, `ColumnGraph`, `BarGraph`, `ScatterGraph` with `data={m.points}`, `xField`, `yField` (default `x` and `y`).
- **Individual elements** are repeated per record: `Column`, `Bar`, `Marker`, `PieSlice` inside a `Repeater`, bound to the record alias. Use them when elements differ from each other — their own colour, tooltip or selection.

```tsx
<PieChart>
  <Repeater records={m.categories} recordAlias={m.$category}>
    <PieSlice name={m.$category.name} value={m.$category.amount} colorMap="categories" r={80} r0={50} />
  </Repeater>
</PieChart>
```

- Columns side by side: give each series `size` and `offset`, as fractions of the category width — `size={0.3}` with `offset={-0.15}` and `offset={0.15}`.
- Stacking: `stacked` on each series; `stack="name"` keeps separate stacks apart.
- Tooltips on chart elements need `enableTooltips()`.

## Colours

- **Values with a meaning get a fixed `colorIndex`** (0–15 from the theme palette) — turnover always blue, expenses always red, in every chart.
- **Series not known up front** — one per category, loaded from data — get their colours from a `ColorMap`: place `<ColorMap />` inside the `Svg` and give each element the same `colorMap="name"`. The map spreads its colours as far apart as possible.
- `ColorMap.Scope` limits a colour map to part of the page.

## Legends

`Legend` is HTML, not SVG — place it outside the `Svg`. Every element with a `name` reports itself to the legend. Clicking an entry toggles the series if its `active` prop is bound to a writable path: `active={bind(m.showTurnover, true)}`.

`Legend.Scope` groups elements and legends. **Charts that show the same categories share one scope** and one legend — turnover and expenses across several charts appear once and toggle together. Charts with unrelated series each get their own scope, so their entries do not mix.

## Documentation

| Task | Page |
|---|---|
| basics | [Overview](https://cxjs.io/docs/charts/overview.md), [Svg](https://cxjs.io/docs/charts/svg.md), [Chart](https://cxjs.io/docs/charts/chart.md) |
| axes | [NumericAxis](https://cxjs.io/docs/charts/numeric-axis.md), [CategoryAxis](https://cxjs.io/docs/charts/category-axis.md), [TimeAxis](https://cxjs.io/docs/charts/time-axis.md) |
| series | [LineGraph](https://cxjs.io/docs/charts/line-graph.md), [ColumnGraph](https://cxjs.io/docs/charts/column-graph.md), [BarGraph](https://cxjs.io/docs/charts/bar-graph.md), [ScatterGraph](https://cxjs.io/docs/charts/scatter-graph.md), [PieChart](https://cxjs.io/docs/charts/pie-chart.md) |
| legend and colours | [Legend](https://cxjs.io/docs/charts/legend.md), [ColorMap](https://cxjs.io/docs/charts/color-map.md) |
| annotations | [MarkerLine](https://cxjs.io/docs/charts/marker-line.md), [Range](https://cxjs.io/docs/charts/range.md), [Marker](https://cxjs.io/docs/charts/marker.md) |
| interaction | [MouseTracker](https://cxjs.io/docs/charts/mouse-tracker.md), [SnapPointFinder](https://cxjs.io/docs/charts/snap-point-finder.md), [HoverSync](https://cxjs.io/docs/charts/hover-sync.md) |
