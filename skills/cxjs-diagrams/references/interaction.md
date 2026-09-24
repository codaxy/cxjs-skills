# Interaction

## Zoom and pan

```tsx
<Diagram center showGrid zoom={m.view.zoom} offsetX={m.view.offsetX} offsetY={m.view.offsetY}>
```

- The wheel zooms around the cursor, by `zoomStep` (default `0.05`, i.e. 5%) per notch; pinch zooms on touch screens. Dragging with the left button pans.
- `minZoom` / `maxZoom` (default `0.25` / `4`) limit what the user can reach; values written into the bound paths are not clamped.
- `fixed` turns zooming and panning off — for static, embedded diagrams.
- Bind `zoom`, `offsetX` and `offsetY` to keep the view when the user comes back — in page data, the view survives navigation between tabs of the same page.

What the numbers mean:

- `offsetX`/`offsetY` are **pixels from the centre of the diagram's box**. With `0`, the layout's origin — the top-left corner of the outermost `Flow` — sits in the middle of the view. `center` (or `centerX`/`centerY`) shifts the content so that it is centred instead.
- `zoom` is a plain scale factor.

While the user drags or zooms, the diagram keeps the view internally and writes it to the store after a 100ms pause. The bound values lag behind an interaction in progress; do not drive other logic from them in real time.

### Centring

`center` centres the content on every render. That is right for a diagram whose content does not change while it is shown. When the content changes — nodes are added, a branch opens — while the user keeps a pan, everything shifts under the held offset. For such diagrams leave `center` off and set `offsetX`/`offsetY` yourself, once, when the diagram first appears.

## Click and context menu

Shapes take `onClick`, `onDoubleClick` and `onContextMenu` with `(e, instance)` — see [shapes-and-lines.md](shapes-and-lines.md#events). For a context menu, use CxJS `openContextMenu(e, <Menu>…</Menu>, instance)` and call `e.preventDefault()`.

## Selection

A `Shape` becomes selectable with a `selection` configuration — the same model as CxJS `Grid` and `List`. Pass the record so the selection can identify the shape:

```tsx
<Repeater records={m.nodes} recordAlias={m.$node}>
  <Cell w={4} h={2}>
    <Shape
      id={m.$node.id}
      text={m.$node.name}
      selection={{ type: KeySelection, bind: m.selectedIds, keyField: "id", multiple: true, record: m.$node }}
    />
  </Cell>
</Repeater>
```

- Click selects; Ctrl+click toggles; Shift+click adds to a multiple selection.
- A selected shape gets the `cxs-selected` state class — style it from there.
- Shapes bound to the same selection path share one selection, so a grid and a diagram can select together.

## Drag and drop between shapes

A shape becomes a drag source with `dragSource` — the data passed to the drop target — and a drop target with `onDrop`:

```tsx
<Shape
  id={m.$node.id}
  text={m.$node.name}
  dragSource={{ id: m.$node.id }}
  onDropTest={(e) => e.source.data.id != null}
  onDrop={(e, instance) =>
    instance.getControllerByType(Controller).link(e.source.data.id, instance.store.get(m.$node.id))
  }
  overShapeClass="node-shape--drop-over"
  farShapeClass="node-shape--drop-target"
/>
```

- `onDropTest` returns `false` to refuse a source; `onDragStart` returns `false` to cancel a drag.
- `overShapeClass`/`overShapeStyle` apply while the cursor is over a target, `farShapeClass`/`farShapeStyle` to all targets during a drag.
- `clone` configures what follows the cursor; by default it is a copy of the shape.
- A shape being dragged gets the `cxs-dragged` state class.

## Moving parts of the diagram: `Draggable`

`Draggable` lets the user move its children. Its `offsetX`/`offsetY` are pixels; bind them to keep the position:

```tsx
<Flow direction="right" gap={2}>
  <Draggable offsetX={m.$node.dx} offsetY={m.$node.dy}>
    <Cell w={3} h={2}><Shape id={m.$node.id} text={m.$node.name} /></Cell>
  </Draggable>
</Flow>
```

- The layout is computed first and the offset applied afterwards: moving one element does not move its siblings, and lines follow the moved shapes.
- Wrap several cells in one `Draggable` to move them as a group.

## Documentation

| Topic | Page |
|---|---|
| zoom, pan, grid | [Diagram](https://diagrams.cxjs.io/components/diagram.md) |
| selection | [Selection](https://diagrams.cxjs.io/examples/selection.md) |
| moving elements | [Draggable](https://diagrams.cxjs.io/components/draggable.md) |
