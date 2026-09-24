# Building diagrams from data

## Build a graph model, render it with Repeaters

Turn the domain data into a small graph model in the controller, write it into the store once, and render it declaratively:

```ts
// model.ts
export interface DiagramNode {
  id: string; // unique across the whole diagram
  text: string;
  kind: "task" | "decision" | "terminal";
}

export interface DiagramEdge {
  from: string;
  to: string;
  label?: string;
}

interface PageModel {
  diagram: { nodes: DiagramNode[]; edges: DiagramEdge[] };
  $node: DiagramNode;
  $edge: DiagramEdge;
}

export default createModel<PageModel>();
```

```tsx
<Svg style="width: 100%; height: 600px">
  <Diagram center>
    <Flow direction="down" gap={2} align="center">
      <Repeater records={m.diagram.nodes} recordAlias={m.$node} keyField="id">
        <Cell width={expr(m.$node.kind, (k) => (k == "decision" ? 6 : 8))} h={2}>
          <Shape
            id={m.$node.id}
            text={m.$node.text}
            shape={expr(m.$node.kind, (k) => (k == "decision" ? "rhombus" : k == "terminal" ? "circle" : "rectangle"))}
            shapeClass="node-shape"
          />
        </Cell>
      </Repeater>
    </Flow>
    <Repeater records={m.diagram.edges} recordAlias={m.$edge}>
      <StraightLine from={m.$edge.from} to={m.$edge.to} class="edge">
        <ArrowHead class="edge-arrow" />
      </StraightLine>
    </Repeater>
  </Diagram>
</Svg>
```

- Build the model with plain functions — no widgets — so it can be tested on its own.
- Keep widget structure out of loops in JSX: the `Repeater`s do the iteration.
- Put the edges **after** the layout, so every shape is registered before a line looks it up.
- Nodes that differ only in size, shape or colour stay one template with bound props — `width`, `height`, `shape` and the classes are all bindable. Use separate templates with `visible` only when the structure really differs, such as extra children or a different click handler.

## Ids

There is one registry per `Diagram`. When the data is nested — lanes, sub-processes, groups — the same local key can appear twice, so build ids from the path:

```ts
const id = `${lane.id}/${task.key}`;
```

For synthetic nodes without a natural key, a counter is fine: `` `connector-${n++}` ``.

## Connection points

A near-zero-size cell with a shape is a connection point: lines can attach to it, but it is invisible.

```tsx
<Cell w={0.01} h={0.01}>
  <Shape id={m.$lane.startId} shape="circle" />
</Cell>
```

Use them where many lines fan out or join — the entry and exit of a lane, the merge point after parallel branches — so an N-to-M join needs N + M lines instead of N × M.

## Nested structures

Lanes of nodes, branches inside nodes: render each level with its own `Repeater` and its own record alias (`$lane`, `$node`, `$branch`), declared in the model. A branching structure is a horizontal `Flow` of vertical `Flow`s:

```tsx
<Flow direction="right" gap={3} align="start">
  <Repeater records={m.diagram.lanes} recordAlias={m.$lane} keyField="id">
    <Flow direction="down" gap={1} align="center">
      <Repeater records={m.$lane.nodes} recordAlias={m.$node} keyField="id">
        <Cell w={8} h={2}><Shape id={m.$node.id} text={m.$node.text} /></Cell>
      </Repeater>
    </Flow>
  </Repeater>
</Flow>
```

When edges belong to a node — each node lists its predecessors — render them right after the node, inside the same `Repeater`. The node's shape is then registered by the time its incoming lines look for it; the predecessors come earlier in the tree.
