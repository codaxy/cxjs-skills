---
name: cxjs-diagrams
description: Use when building or debugging node-and-edge diagrams with the `cx-diagrams` package in a CxJS app — flowcharts, workflow graphs, network topologies, org charts — or when working with `Diagram`, `Flow`, `Cell`, `Shape`, `StraightLine`, `TwoSegmentLine`, `ThreeSegmentLine`, `ArrowHead`, `Rotate`, `FourSides` or `Draggable`.
---

# CxJS Diagrams

`cx-diagrams` draws node-and-edge diagrams inside a CxJS `Svg`. **You never position anything by coordinates.** You nest `Flow` containers and `Cell` boxes on a unit grid, give the shapes ids, and connect them by id. The library computes every position, and lines find their own attachment points.

It builds on CxJS — bindings, typed models, controllers, `Repeater`. If the `cxjs` skill is available, follow its conventions; this skill covers only the diagram layer.

## Where to look

1. **The project itself** — its own documented conventions and the code around the change.
2. **This skill and its references.**
3. **The documentation** — every page is available as Markdown at `https://diagrams.cxjs.io/<section>/<page>.md`; `https://diagrams.cxjs.io/llms.txt` indexes them.
4. **The source**, for exact props and behaviour — the package ships it: `node_modules/cx-diagrams/src/<Widget>.tsx` holds the `XxxConfig` interface, and the defaults are set on the prototype after the class (`Diagram.prototype.unitSize = 32`).

## Setup

- Install `cx-diagrams` next to `cx`.
- Import the base styles once, in the application's SCSS entry: `@use "cx-diagrams/src/index.scss";`. Without them shape labels are not centred and shapes have no pointer cursor.
- The package is an ES module with extensionless internal imports. webpack needs `{ test: /\.m?js$/, resolve: { fullySpecified: false } }`, or it cannot resolve them.

## A minimal diagram

```tsx
import { ArrowHead, Cell, Diagram, Flow, Shape, StraightLine } from "cx-diagrams";
import { Svg } from "cx/svg";

export default (
  <Svg style="width: 100%; height: 400px">
    <Diagram center showGrid>
      <Flow direction="down" gap={2} align="center">
        <Cell w={6} h={2}>
          <Shape id="start" text="Start" fill="white" stroke="gray" />
        </Cell>
        <Cell w={6} h={2}>
          <Shape id="finish" text="Finish" fill="white" stroke="gray" />
        </Cell>
      </Flow>
      <StraightLine from="start" to="finish" stroke="gray">
        <ArrowHead fill="gray" />
      </StraightLine>
    </Diagram>
  </Svg>
);
```

## Rules

| # | Rule | Why |
|---|---|---|
| 1 | **Connections come after the shapes they reference**, in tree order. | Shapes register their bounds while rendering; a line that comes first finds nothing and draws a zero-length line at the top-left corner — and an `ArrowHead` inside it throws `ArrowHead must be placed inside a Line component`. |
| 2 | **Shape ids are unique across the whole `Diagram`**, not per `Flow`. Prefix ids built from nested data: `` `${lane.id}/${task.id}` ``. | There is one registry per diagram; a duplicate id overwrites the earlier one and lines attach to the wrong shape. |
| 3 | **Know the two units.** `Cell` sizes, `Flow` gaps and paddings, and the margins of `Cell`/`Flow` are **grid units** (× `unitSize`, 32px by default). `anchors`, `offset` and `margin` on a `Shape` or any SVG element are **pixels**. | `<Cell w={200}>` is 6400px wide; `<Shape margin={-4}>` inflates a shape by 4px. |
| 4 | `shape` is `rectangle` (default), `circle` or `rhombus`. | Any other value — `"rect"` — draws no outline at all, only the text. |
| 5 | **Give shapes a fill and a stroke**, through props or classes. | The diagram sets `fill: none; stroke: none`, and shapes inherit it. |
| 6 | **The `Svg` needs a real size** — a CSS height, or a flex layout that gives it one. | `Diagram` fills its `Svg`; an `Svg` without height renders nothing. |
| 7 | **Some layout props are not bindable** — `Flow` `direction`, `align`, `justify`, `selfAlign`, and `Diagram` `center`/`centerX`/`centerY`. | They are read once, when the widget is created. Change layout by rendering a different structure. |

## Widgets

| Widget | Role | Key props |
|---|---|---|
| `Diagram` | viewport: zoom, pan, grid; holds the shape registry | `zoom`, `offsetX`, `offsetY` (bind them to keep the viewport), `unitSize`, `showGrid`, `fixed`, `center`, `zoomStep`, `minZoom`, `maxZoom` |
| `Flow` | lays out its children in one direction | `direction` (`right` default, `left`, `up`, `down`), `gap`, `p`/`px`/`pt`…, `align` (`start` default, `center`), `justify="space-between"`, `selfAlign="stretch"` |
| `Cell` | a fixed-size box on the grid | `w`/`width`, `h`/`height` (default `1`) |
| `Shape` | a drawn node | `id`, `text`, `shape`, `fill`, `stroke`, `strokeWidth`, `shapeClass`, `textClass`, `tooltip`, `onClick`, `selection`, drag & drop props |
| `StraightLine` | straight connection | `from`, `to`, `stroke`, `class`, `style` |
| `TwoSegmentLine` / `ThreeSegmentLine` | L- and Z-shaped connections | `from`, `to`, `direction`, `startOffset` |
| `ArrowHead` | marker on the line that contains it | `position` (`end` default, `start`, `middle`), `shape` (`triangle` default, `vback`, `line`), `size` (default `12`) |
| `Rotate` | rotates flow and line directions below it | `turns` (× 90° clockwise) |
| `FourSides` | places children around a centre | `gap`, `slots` (default `["center", "right", "down", "left", "up"]`) |
| `Draggable` | lets users move its children | `offsetX`, `offsetY` in pixels — bind them to keep the position |

All layout widgets — `Cell`, `Flow`, `Rotate`, `FourSides`, `Draggable` — take the margin props `m`, `mx`/`my`, `ml`/`mr`/`mt`/`mb`, and `ms`/`me` (start and end in the flow direction), plus `grow`, `msAuto` and `meAuto` to share free space.

## Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| a line is missing, or drawn from the top-left corner | its `from`/`to` shape is not registered yet, or the id is wrong | put connections after the layout; check the ids match |
| `ArrowHead must be placed inside a Line component` | the arrow is outside a line, or inside a line whose ids do not resolve | nest it in a line; fix the ids |
| lines attach to the wrong shapes | duplicate ids | make ids unique across the diagram |
| a shape draws only its text | `shape="rect"`, or no fill and stroke | use `rectangle`; set `fill`/`stroke` or classes |
| everything piles up in one place | `Cell`s not inside a `Flow` | wrap them in a `Flow` |
| lines are drawn over the shapes | SVG paints in document order, and lines come after the shapes | render the lines into a `ContentPlaceholder` placed before the shapes — see [shapes-and-lines.md](references/shapes-and-lines.md#drawing-lines-below-shapes) |
| an elbow line does not meet a circle or rhombus on its outline | two- and three-segment lines attach to the middle of the bounding box's sides | expected; use `StraightLine` when the edge should meet the outline |
| the diagram is invisible | the `Svg` has no height | give it a real size |
| labels are not centred, no pointer cursor | the base stylesheet is not loaded | `@use "cx-diagrams/src/index.scss";` |

## References

Read the file that matches the task.

| File | Covers |
|---|---|
| [references/layout.md](references/layout.md) | the unit grid, how `Flow` places children, alignment, sharing free space, margins and padding, `Rotate`, `FourSides`, positioning with `anchors` |
| [references/shapes-and-lines.md](references/shapes-and-lines.md) | `Shape`, the three line types, children of lines, `ArrowHead`, styling, tooltips |
| [references/interaction.md](references/interaction.md) | zoom and pan, keeping the viewport, click handlers, selection, drag & drop, `Draggable` |
| [references/data-driven.md](references/data-driven.md) | building diagrams from data: the graph model, ids, connection points, edges, nodes that differ, nested structures |
