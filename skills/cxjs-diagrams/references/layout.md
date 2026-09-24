# Layout

## The unit grid

Layout sizes are **grid units**, converted to pixels by `Diagram unitSize` (default `32`):

- `<Cell w={6} h={2}>` is 192×64px at the default unit size.
- Changing `unitSize` rescales the whole layout. It is the design scale; `zoom` is the user's view.
- Fractional units are fine: `<Cell w={0.01} h={0.01}>` is a near-zero-size connection point (see [data-driven.md](data-driven.md)).

Pick a unit that makes typical nodes small integers — `w={6} h={2}` for a node, gaps of `1`–`3`.

## How `Flow` places children

`Flow` walks its children in order and advances along `direction` (`right` by default):

- each child takes `ms` + its margin + its size + its margin + `me`, and `gap` separates consecutive children;
- the flow's size on the cross axis is its largest child plus padding;
- the resulting box becomes a child of the enclosing flow, so the whole layout is computed bottom-up.

**Only layout widgets take a slot**: `Cell`, `Flow`, `Rotate`, `FourSides`, `Draggable`. Everything else — `Shape`, and `Rectangle`, `Text`, `Line` from `cx/svg` — is positioned inside the box of its parent. That is what makes a `Rectangle` the background of a `Flow`:

```tsx
<Flow direction="down" gap={1} p={1}>
  <Rectangle fill="#f3f4f6" stroke="#d1d5db" />
  <Text value={m.$lane.name} anchors="0 0 0 0" offset="8 8 8 8" dy="0.8em" />
  …the lane's nodes…
</Flow>
```

## Alignment and free space

- `align="center"` centres children on the cross axis; the default is `start`.
- `justify="space-between"` spreads free space between children along the main axis.
- `selfAlign="stretch"` on a child `Flow` makes it fill the parent's cross axis. It exists only on `Flow`, not on `Cell`.
- `grow` on a child makes it take the free space along the flow; `msAuto`/`meAuto` turn free space into a start or end margin — both together centre the child. Growing children are filled first, the auto margins get the rest.

Free space exists only when a flow is larger than its content — typically because it is stretched by `selfAlign="stretch"`:

```tsx
<Flow direction="down" gap={1}>
  <Flow direction="right" gap={1}>…three cells set the width…</Flow>
  <Flow direction="right" selfAlign="stretch">
    <Cell w={2} grow>…fills the row…</Cell>
  </Flow>
  <Flow direction="right" selfAlign="stretch">
    <Cell w={2} msAuto>…pushed to the end…</Cell>
  </Flow>
</Flow>
```

A column whose first and last cells stay pinned to the ends while the rest flows between them:

```tsx
<Flow direction="down" gap={1} align="center" selfAlign="stretch" justify="space-between">
  <Cell w={0.01} h={0.01}><Shape id={m.$lane.startId} shape="circle" /></Cell>
  <Flow direction="down" gap={1} align="center">…nodes…</Flow>
  <Cell w={0.01} h={0.01}><Shape id={m.$lane.endId} shape="circle" /></Cell>
</Flow>
```

## Margins and padding

- Padding (`p`, `px`/`py`, `pl`/`pr`/`pt`/`pb`) is on `Flow` only — space inside it, around its children.
- Margins (`m`, `mx`/`my`, `ml`/`mr`/`mt`/`mb`) are on every layout widget — space outside it, between it and its siblings. Specific values win: `ml` over `mx` over `m`.
- `ms`/`me` are the start and end margins *in the flow direction*. Prefer them inside a `Rotate`, where left and right no longer mean what they say.

## `Rotate` and `FourSides`

`Rotate turns={n}` remaps the flow and line directions below it by n × 90° clockwise — a `Flow direction="right"` inside `<Rotate turns={1}>` runs downward. It does **not** rotate graphics: shapes and text stay upright. Use it to lay out a structure once and place it in four orientations. A `Flow` with `fixed` keeps its own direction. `turns` must not be negative.

`FourSides` places its children around a centre in the order of `slots` (default `["center", "right", "down", "left", "up"]`). Pass a shorter list to use fewer sides — with two groups, `["right", "left"]` puts them opposite each other. Children beyond the number of slots are not placed.

```tsx
<FourSides gap={2} slots={["center", "right", "left"]}>
  <Cell w={4} h={2}><Shape id="core" text="Core" /></Cell>
  <Flow direction="down" gap={1}>…right group…</Flow>
  <Flow direction="down" gap={1}>…left group…</Flow>
</FourSides>
```

## Positioning inside a shape or a line

Inside a `Cell`, `Shape` or line, SVG elements and nested shapes position themselves with `anchors`, `offset` and `margin`, in **pixels** — the same mechanism as CxJS charts:

- `anchors` place each edge at a fraction of the parent box, `"top right bottom left"`;
- `offset` then adds pixels to each edge (positive moves right or down);
- `margin` works like CSS margin — positive shrinks, negative grows.

| Want | Write |
|---|---|
| fill the parent — the default for `Shape` | `anchors="0 1 1 0"` |
| a 16×16 badge in the centre | `anchors="0.5 0.5 0.5 0.5" offset="-8 8 8 -8"` |
| a 20×20 badge in the top-right corner, 2px inside | `anchors="0 1 0 1" offset="2 -2 22 -22"` |
| a 6px connection dot on the middle of the top edge | `anchors="0 0.5 0 0.5" margin={-3}` |
| a strip along the right edge, 26px wide | `anchors="0 1 1 1" offset="2 -2 -2 -28"` |

A small shape placed this way can carry its own `id`, which makes it a connection point on its parent's edge.

## Sanity checks

- Nodes pile up in one place → they are not inside a `Flow`.
- A group drifts away from its neighbours → a margin that should have been padding, or `align` missing.
- The layout jumps between refreshes → a cell appears and disappears; keep the cell and toggle its contents instead.
