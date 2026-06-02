# splose Widget Shape Generator

A parametric SVG shape for the brand direction: a body rectangle with up to two
independent diagonal "kicks" where the top and/or bottom edge steps up at a fixed
angle. Built to be responsive (scales via `viewBox`) and framework-agnostic.

The reference artifact is `index.html` — a standalone interactive
playground with live sliders and copy buttons. The core logic lives in two pure
functions, `sploseShapePath()` and `roundedPolyPath()`, which are the parts to lift
into the codebase.

## The shape, conceptually

Start with a plain rectangle (the "body"). Each horizontal edge can optionally
"kick up" — step to a greater height — via a straight diagonal segment held at a
fixed angle to the horizontal (50° is the brand default). The top edge and bottom
edge each have their own kick, fully independent of one another:

- Both kicks zero → plain rounded rectangle.
- One kick zero → single chamfer on the other edge.
- Both kicks present at different x-positions → the signature "two boxes joined by
  a diagonal bridge" look. A tall middle region appears naturally wherever the
  top edge has already kicked up but the bottom edge has not yet (or vice versa).

Only the corners are rounded (quadratic beziers). Every diagonal edge stays a
perfectly straight line at the chosen angle — the rounding is applied at vertices
only and never bends the straight runs.

## Geometry rules (do not break these)

- **Fixed angle.** Diagonals are locked to `angle` degrees from horizontal.
  The horizontal run of a kick is derived, never set directly:
  `run = kickAmount / tan(angle)`. Changing the kick height changes the run so
  the angle is preserved.
- **Top-edge alignment.** All measurements are taken from a common top reference.
  Heights grow downward; kicks raise an edge by moving it *up* (toward smaller y).
- **Horizontal offsets only.** Kick positions are expressed as a percentage of
  total width (`topKickPos`, `bottomKickPos`), i.e. where along the x-axis each
  diagonal is centered. The bridge travels horizontally, not vertically.
- **Independent kicks.** The top and bottom diagonals are decoupled — they can sit
  at different x-positions and have different amounts. This is required: the design
  calls for e.g. the top edge kicking up around 20% width while the bottom edge
  kicks up around 80% width.
- **Kick position is clamped** so the diagonal's run fits inside the width:
  `clamp(pos% * W, run/2, W - run/2)`.
- **Zeroable.** When a kick amount is at/below the epsilon threshold (0.5), its two
  intermediate vertices are omitted entirely, leaving a flat edge. This is what
  lets the shape degrade gracefully to a chamfer or a plain rounded rectangle.

## Function API

### `sploseShapePath(opts) → { d, width, height, topCenter, bottomCenter }`

Returns the SVG path `d` string plus the natural dimensions to feed the `viewBox`.

| option          | default | meaning                                                        |
|-----------------|---------|----------------------------------------------------------------|
| `width`         | 360     | total horizontal extent in world units                         |
| `baseHeight`    | 110     | body height at the left edge                                   |
| `topKick`       | 125     | how far the top edge steps up; `0` = flat top, no diagonal     |
| `bottomKick`    | 65      | how far the bottom edge steps up; `0` = flat bottom            |
| `topKickPos`    | 22      | % of width where the top kick is centered                      |
| `bottomKickPos` | 78      | % of width where the bottom kick is centered                   |
| `radius`        | 14      | corner radius (auto-limited per-vertex so it never overshoots) |
| `angle`         | 50      | diagonal angle in degrees from horizontal                      |

Returned `topCenter` / `bottomCenter` are the resolved (clamped) x-positions of
each kick — handy for drawing guides or aligning other elements.

The polygon is assembled top-edge-first, clockwise: top-left → (optional top kick
vertices) → top-right → right edge down → (optional bottom kick vertices) →
bottom-left → close. After building, the whole point set is shifted so the
tallest point sits at `y = 0` and `height` reports the full extent.

### `roundedPolyPath(pts, r) → string`

Generic helper: takes an array of `[x, y]` points for a closed polygon and returns
a path string with every vertex rounded by radius `r` using quadratic beziers. The
radius is clamped per-vertex to half the shorter adjacent edge, so tight corners
and short segments stay valid. This is deliberately generic — it has no knowledge
of the splose shape and could round any polygon.

## Rendering

```js
const { d, width, height } = sploseShapePath({ baseHeight: 110, topKick: 125, bottomKick: 65 });
const pad = 20; // optional breathing room
// viewBox="-pad -pad width+2pad height+2pad" keeps it responsive at any size
```

Set `width="100%"` on the `<svg>` and let `viewBox` handle all scaling — no media
queries needed. The fill currently used is the 'new' brand purple `#a78fe0`.

## Next steps / open work

- **TypeScript component.** Wrap `sploseShapePath` as a typed `.tsx` component with
  the options as props, for the Vite app. Keep the two functions as a pure util
  module (e.g. `brandShape.ts`) so they stay testable and SSR-safe.
- **Token the constants.** `angle` (50), brand fill, and default radius should come
  from design tokens rather than literals once integrated.
- **Variants.** Consider exposing a flipped/mirrored variant (kicks going down
  instead of up) if the brand system needs the inverse silhouette.
- **Verify angle under rounding.** The straight middle of each diagonal is exactly
  `angle°`; corner rounding shortens the ends but does not change the slope. If
  very large radii are combined with very short kicks, sanity-check visually.

## Stack notes

Project uses JavaScript/TypeScript with Vite. The generator functions are plain
ES with no dependencies and drop in as-is.
