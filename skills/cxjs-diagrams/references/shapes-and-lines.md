# Shapes and lines

## `Shape`

A `Shape` fills its parent box by default (`anchors="0 1 1 0"`) — usually a `Cell`. It renders as `<g id={id}>` containing the outline, the text and any children.

- `id` registers the shape for connections; it is required on any shape a line points at.
- `shape`: `rectangle` (default), `circle`, `rhombus`. A circle uses the smaller of width and height as its diameter, so circular nodes want a square cell.
- `text` is drawn centred. For richer content, add children — they are positioned against the shape's box with `anchors`/`offset` (see [layout.md](layout.md)).
- `tooltip` takes a string or a CxJS tooltip configuration; tooltips need `enableTooltips()` at startup.

### Styling

Styles land on three different elements:

| Prop | Lands on | Use for |
|---|---|---|
| `fill`, `stroke`, `strokeWidth` | the outline | quick colours |
| `shapeClass` / `shapeStyle` | the outline (`rect`, `ellipse` or `polygon`) | fill, stroke, filters |
| `textClass` / `textStyle` | the text | colour, font, weight |
| `class` | the wrapping `<g>` | state hooks that must reach both outline and text, such as hover |

`style` goes on the `<g>`, and is also applied to the outline when `shapeStyle` is not set — prefer `shapeStyle` for the outline.

```tsx
<Shape
  id={m.$node.id}
  text={m.$node.name}
  class="node"
  shapeClass={{ "node-shape": true, "node-shape--failed": equal(m.$node.status, "failed") }}
  textClass="node-text"
/>
```

```scss
.node:hover .node-shape { filter: brightness(0.95); }
.node-shape { fill: white; stroke: #9ca3af; }
.node-shape--failed { stroke: #dc2626; }
```

Built-in classes: `cxb-shape` on the `<g>`, `cxe-shape-shape` on the outline, `cxe-shape-text` on the text, and the states `cxs-selected` and `cxs-dragged`. Prefer your own classes over overriding the built-in ones — every diagram in the application shares them.

SVG elements take `drop-shadow()` in `filter` for elevation; `box-shadow` does not apply to them.

### Events

`onClick`, `onDoubleClick` and `onContextMenu` receive `(e, instance)`. Call controller methods through `getControllerByType`, and read the node's record from the instance:

```tsx
<Shape
  id={m.$node.id}
  text={m.$node.name}
  onClick={(e, instance) =>
    instance.getControllerByType(Controller).openNode(instance.store.get(m.$node.id))
  }
/>
```

`Shape` has no `onMouseDown` prop.

## Lines

All lines take `from` and `to` — the ids of two shapes — and must come after both shapes in the tree.

| Line | Route | Where it meets the shapes |
|---|---|---|
| `StraightLine` | one segment, centre to centre | on the outline — a circle's circumference, a rhombus's edge, a rectangle's side |
| `TwoSegmentLine` | L-shaped | the middle of a side of the bounding box |
| `ThreeSegmentLine` | Z-shaped, with the middle segment halfway | the middle of a side of the bounding box |

- `direction` (`right`, `left`, `up`, `down`) sets the direction of the first segment of an elbow line.
- `startOffset` (pixels) shifts where an elbow line leaves the start shape, across its first segment — use it to keep parallel lines from overlapping.
- `stroke` sets the colour. Lines have no `strokeWidth` prop — set the width with `style="stroke-width: 2px"` or a class.
- `class` lands on the line's `<g>`, so it styles every segment of an elbow at once; `style` and `stroke` land on each segment. Each segment is its own `<line>` element.

### Drawing lines below shapes

SVG paints in document order, so lines that come after the shapes — as rule 1 requires — are drawn on top of them. To keep that order for layout but paint the lines underneath, put a `ContentPlaceholder` before the shapes and send the lines into it:

```tsx
<Diagram center>
  <ContentPlaceholder name="lines" />
  <Flow direction="down" gap={2}>…shapes…</Flow>
  <PureContainer putInto="lines">
    <Repeater records={m.diagram.edges} recordAlias={m.$edge}>
      <StraightLine from={m.$edge.from} to={m.$edge.to} class="edge" />
    </Repeater>
  </PureContainer>
</Diagram>
```

The lines are still processed where they are declared — after the shapes, so they find them — and only their output is rendered at the placeholder. A placeholder renders **one** piece of content: wrap all lines in a single container with `putInto`, as above, or set `allowMultiple` on the placeholder. `ContentPlaceholder` and `PureContainer` come from `cx/ui`.

### Children of a line

A line's box runs from its start point (top-left) to its end point (bottom-right). Children position along it:

```tsx
<StraightLine from={m.$edge.from} to={m.$edge.to} class="edge">
  <Shape anchors="0 0 0 0" margin={-4} shape="circle" fill="white" stroke="gray" />   {/* start */}
  <Shape anchors="1 1 1 1" margin={-4} shape="circle" fill="white" stroke="gray" />   {/* end */}
  <Text value={m.$edge.label} anchors="0.5 0.5 0.5 0.5" textAnchor="middle" dy="-0.4em" />   {/* middle */}
</StraightLine>
```

A label on a line stays readable with a halo: `paint-order: stroke; stroke: white; stroke-width: 6px; stroke-linejoin: round`.

## `ArrowHead`

An `ArrowHead` sits inside the line it decorates:

```tsx
<ThreeSegmentLine from={m.$edge.from} to={m.$edge.to} direction="down" class="edge">
  <ArrowHead position="end" shape="triangle" size={9} class="edge-arrow" />
</ThreeSegmentLine>
```

- `position`: `end` (default), `start`, or `middle` — one per segment.
- `shape`: `triangle` (default), `vback`, `line`. `triangle` and `vback` are **filled** — colour them with `fill`, not `stroke`. `line` has no area and needs a `stroke`.
- `size` in pixels (default `12`), `aspectRatio` (default `1`), `reverse` to flip it.
- `class` and `style` land on each arrow `<path>`.
- Outside a line, or in a line whose shapes do not resolve, it throws `ArrowHead must be placed inside a Line component`.

## Documentation

| Topic | Page |
|---|---|
| shapes | [Shape](https://diagrams.cxjs.io/components/shape.md) |
| lines | [StraightLine](https://diagrams.cxjs.io/lines/straight-line.md), [TwoSegmentLine](https://diagrams.cxjs.io/lines/two-segment-line.md), [ThreeSegmentLine](https://diagrams.cxjs.io/lines/three-segment-line.md) |
| arrows | [ArrowHead](https://diagrams.cxjs.io/components/arrow-head.md) |
