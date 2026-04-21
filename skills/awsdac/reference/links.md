# Links reference

```yaml
Links:
  - Source: <ResourceName>          # required
    SourcePosition: NNE             # optional; default auto
    Target: <ResourceName>          # required
    TargetPosition: S               # optional; default auto
    Type: orthogonal                # optional; default straight
    LineWidth: 1                    # optional
    LineColor: 'rgba(r,g,b,a)'      # optional
    LineStyle: normal               # normal | dashed
    SourceArrowHead:                # optional
      Type: Open                    # Open | Default
      Width: Default                # Narrow | Default | Wide
      Length: 2
    TargetArrowHead:
      Type: Open
    Labels:                         # optional
      SourceLeft:  { Title: "..." }
      SourceRight: { Title: "..." }
      TargetLeft:  { Title: "..." }
      TargetRight: { Title: "..." }
      AutoLeft:    { Title: "..." }  # orthogonal only
      AutoRight:   { Title: "..." }
```

## Position values (16-wind rose)

`N, NNE, NE, ENE, E, ESE, SE, SSE, S, SSW, SW, WSW, W, WNW, NW, NNW`

Omit (or set `auto`) to let awsdac pick based on the lowest common ancestor of source and target. Cardinal points (`N, E, S, W`) hit the icon's edge midpoints; intermediate points offset along that edge.

Prefer explicit positions for primary arrows — auto fallback can be surprising when source and target lack a shared ancestor.

## Link types

- **straight** (default): a single line, diagonals allowed.
- **orthogonal** (`Type: orthogonal`): right-angle bends. Single-arm (one bend) or double-arm (two bends) — chosen automatically from source/target positions.

## Arrowheads

Set `SourceArrowHead` and/or `TargetArrowHead`:

- `Type: Open` — open triangle (typical for data flow)
- `Type: Default` — filled
- `Width: Narrow | Default | Wide`
- `Length` — integer length in px units

## Labels

Four slots on straight links: `SourceLeft`, `SourceRight`, `TargetLeft`, `TargetRight`. Orthogonal links add `AutoLeft` / `AutoRight` that attach to the middle arm. Each label is an object: `{Title, Color, Font}`.

## Overlap control

If multiple links share an edge position, set on the parent container:

```yaml
Parent:
  Options:
    GroupingOffset: true
    GroupingOffsetDirection: true
```

To reduce crossings by reordering siblings, set `UnorderedChildren: true` on each level between source and target (including the LCA).
