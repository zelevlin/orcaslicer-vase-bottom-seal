# Vase Bottom Seal Task

## Problem

In spiral vase mode, the side wall is intentionally printed as one continuous wall, while the bottom may contain multiple solid layers. For complex vase bottoms, the existing solid infill patterns can leave small unfilled gaps or islands. These gaps make the bottom non-hermetic even when the spiral wall itself prints cleanly.

## Goal

Add an OrcaSlicer feature that helps force-seal the bottom of spiral-vase prints by adding extra extrusion in detected bottom-layer voids or user-selected risky areas.

The first useful version should focus on the bottom solid layers only. It should not change the spiralized outer wall path, and it should stay disabled by default for backward compatibility.

## Initial Implementation Direction

1. Identify where bottom solid infill is generated for spiral vase mode.
2. Detect small unfilled regions left inside bottom solid areas after the normal infill path generation.
3. Add a configurable sealing pass that emits extra extrusion paths in those regions.
4. Expose settings conservatively, likely under a spiral vase or quality/strength section:
   - Enable bottom sealing pass.
   - Maximum void size to seal.
   - Extra flow or line spacing for seal paths.
   - Limit to first N bottom layers.
5. Add tests around geometry/path generation so the feature can be validated without relying only on visual G-code inspection.

## Proposed Algorithm: Micro Void Seal Pass

The existing gap fill implementation in `src/libslic3r/Fill/FillBase.cpp` is not enough for hermetic vase bottoms. It computes remaining uncovered solid areas after infill, then uses morphology and medial-axis generation:

- `opening_ex(...)` removes very small regions before line generation.
- `ExPolygon::medial_axis(min, max, ...)` needs a viable skeleton path.
- `filter_out_gap_fill = 0` only disables the final length filter; it does not prevent earlier geometry steps from deleting or ignoring micro voids.

For vase-bottom sealing, add a second pass after normal gap fill:

1. Compute the residual uncovered area again after regular solid infill and regular gap fill:
   - target area: `this->no_overlap_expolygons`
   - already covered area: `out->polygons_covered_by_spacing(...)` or `out->polygons_covered_by_width(...)`
   - residual: `diff_ex(target, covered)`
2. Keep only bottom-cap layers in spiral vase mode, with a new disabled-by-default option.
3. Split residual polygons into two classes:
   - normal gaps: continue using existing gap fill.
   - micro voids: regions below the existing medial-axis threshold or below one extrusion line width.
4. For each micro void, emit an intentional sealing extrusion instead of asking medial-axis to fit a valid line:
   - Use the void bounding box center.
   - Emit one short segment through the longest bbox axis.
   - Segment length should be clamped, for example `0.35 * nozzle_width` to `1.25 * nozzle_width`.
   - Use `erGapFill` or a new role later, but start with `erGapFill` to reuse existing preview, speed, and flow handling.
   - Apply a configurable flow multiplier, for example 120-180%, because the goal is sealing, not perfect dimensional accuracy.
5. Optionally add a cross-stitch mode for rounder voids:
   - one stroke along the longest axis
   - one shorter perpendicular stroke
   - only if the area is still above a minimal threshold

This deliberately turns invisible/ignored geometric leftovers into printable plastic strokes. It may slightly overfill local spots, but for hidden vase bottoms that tradeoff is acceptable and configurable.

## First Option Set

- `vase_bottom_micro_void_seal`: bool, default false.
- `vase_bottom_micro_void_max_area`: float in mm², default `0.20`.
- `vase_bottom_micro_void_flow_ratio`: float, default `1.35`.
- `vase_bottom_micro_void_layers`: int, default `0` meaning all bottom solid layers, otherwise limit to first N layers.

The feature should initially be active only when `spiral_mode = true`. Later it may be generalized to normal bottom surfaces if useful.

## Implemented First Pass

The first implementation lives in `src/libslic3r/Fill/FillBase.cpp`.

After normal gap fill, `_create_vase_bottom_micro_void_seal()`:

1. Requires `spiral_mode` and `vase_bottom_micro_void_seal`.
2. Limits itself to solid infill roles and configured bottom shell layers.
3. Computes residual uncovered regions from `no_overlap_expolygons` minus the actual extrusion-width coverage.
4. Keeps only residual regions under `vase_bottom_micro_void_max_area`.
5. Emits short `erGapFill` sealing strokes through each micro void, using `vase_bottom_micro_void_flow_ratio`.

The first version intentionally reuses `erGapFill` so speed, preview coloring, and existing G-code handling stay compatible.

## Constraints

- Keep existing print behavior unchanged unless the new option is enabled.
- Preserve compatibility with existing 3MF files and printer profiles.
- Keep the implementation cross-platform for macOS, Windows, and Linux.
- Prioritize deterministic geometry output; avoid ad hoc UI-only fixes.

## Open Questions

- Should sealing be fully automatic from detected voids, user-controlled by painted/manual regions, or both?
- Should the first version seal only internal bottom voids, or also allow controlled over-extrusion along bottom perimeters?
- Which G-code visualization metric best proves that previously open regions are now covered?

## Second Feature: Seal Bottom Large Voids (`vase_bottom_gap_fill`)

### Problem

The existing `vase_bottom_micro_void_seal` uses a bounding-box stroke approach designed for sub-medial-axis voids (< 0.20 mm² by default). Larger voids that the standard gap fill still misses — for example narrow gaps where `opening_ex` erodes the geometry before medial-axis can run — are not covered.

When a user enables both the standard OrcaSlicer gap fill and `vase_bottom_micro_void_seal`, the micro-void pass runs after the standard gap fill, so it may lay strokes over already-filled areas. This is harmless but wastes plastic and can over-extrude.

### Goal

Add a dedicated large-void pass that:
1. Targets residual voids **at or above** `vase_bottom_gap_fill_min_area` (default 0.20 mm²) — the boundary where the micro-void pass stops.
2. Uses the full medial-axis gap-fill algorithm (same as `_create_gap_fill`) so it can correctly fill wide regions.
3. Runs **before** the micro-void pass so the two passes cover complementary size ranges without overlap.
4. Has its own configurable flow ratio (default 1.0) and layer limit (0 = all bottom layers).

### Implementation

New function `_create_vase_bottom_gap_fill` in `src/libslic3r/Fill/FillBase.cpp`:

1. Guards: `spiral_mode` + `vase_bottom_gap_fill` enabled, solid infill role, layer within limit.
2. Computes residual = `no_overlap_expolygons` minus `polygons_covered_by_spacing`.
3. Keeps only polygons with `area >= vase_bottom_gap_fill_min_area`.
4. Runs `opening_ex` / `offset2_ex` / `medial_axis` (identical to standard gap fill).
5. Applies `vase_bottom_gap_fill_flow_ratio` by scaling `mm3_per_mm` on each path.
6. Appends to `out` before `_create_vase_bottom_micro_void_seal` is called.

Call order in `_create_gap_fill`:
```
_create_vase_bottom_gap_fill(...)   // large voids first
_create_vase_bottom_micro_void_seal(...)  // micro voids on remaining residual
```

### New Options (Special Mode → Advanced, spiral_mode required)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `vase_bottom_gap_fill` | bool | false | Enable large-void sealing pass |
| `vase_bottom_gap_fill_min_area` | float mm² | 0.20 | Min void area for this pass (set equal to micro-void max area for full coverage) |
| `vase_bottom_gap_fill_flow_ratio` | float | 1.0 | Flow multiplier for the pass |
| `vase_bottom_gap_fill_layers` | int | 0 | Limit to first N bottom layers; 0 = all |

### Key Files Changed (second feature)

- `src/libslic3r/PrintConfig.hpp` — 4 new struct members
- `src/libslic3r/PrintConfig.cpp` — 4 new option definitions
- `src/libslic3r/Fill/FillBase.hpp` — new declaration
- `src/libslic3r/Fill/FillBase.cpp` — implementation + call site
- `src/libslic3r/Preset.cpp` — registration in `print_options()`
- `src/libslic3r/PrintObject.cpp` — `posInfill` invalidation for all 8 new keys
- `src/libslic3r/Print.cpp` — `osteps posInfill` invalidation for all 8 new keys
- `src/slic3r/GUI/Tab.cpp` — UI lines in Special Mode group
- `src/slic3r/GUI/ConfigManipulation.cpp` — visibility toggles

### Residual Coverage in micro-void pass

The micro-void pass (`_create_vase_bottom_micro_void_seal`) computes the uncovered area
via `polygons_covered_by_width(10)` — i.e. it buffers each extrusion by its actual print
width. When the large-void pass runs first and appends medial-axis lines to `out`, those
lines are included in this coverage calculation, so the micro-void pass naturally skips
areas already covered by the large pass.

**Known limitation (2026-05-04):** at the boundary of a large-void fill the micro-void
pass may still emit a short stroke that slightly overlaps the edge of the large-void fill.
This is cosmetically harmless (it just adds a small amount of extra plastic on top of
already-covered geometry) and accepted for now. The overlapping strokes are short and do
not cause visible print defects at typical flow ratios.
