---
name: gee-tile-temporal-mosaic
description: Use when a Google Earth Engine task must build a low-cloud mosaic for an ROI across multiple Sentinel-2 or Landsat tiles while minimizing tile count, coordinating acquisition dates, and avoiding implicit cross-date per-pixel selection.
---

# GEE Tile Temporal Mosaic

## Purpose

Build a spatially complete ROI product from the fewest required tiles while keeping each tile's pixels tied to one selected acquisition. This is a **tile-level temporal mosaic**, not a generic cloud-free composite.

The default product is one selected image per required tile, assembled in a fixed priority order. Do not silently use `qualityMosaic`, cross-date `median`, or any per-pixel search over dates.

The user-facing result is a handoff package, not only a selection explanation:
show the selected scene IDs and quality diagnostics, create a preview, and
provide a short copy-paste script that imports exactly those scenes, reapplies
the same masks, and builds the priority mosaic.

## Backend Gate and Handoff

Before any Earth Engine query or code generation, ask the user to choose one
backend explicitly:

| Choice | Primary artifact | Preview | Runtime requirements |
|---|---|---|---|
| Native GEE Code Editor JavaScript | `selected_mosaic.js` (and, when requested, the full selector) | `Map.addLayer` / `print` in Code Editor | no local Python or geemap |
| Python/geemap | `code.py` plus a compact handoff block | geemap HTML and/or PNG | local Python, geemap, credentials, Cloud Project ID |
| Both | both artifact sets from the same selected combination | both previews | satisfy both sets of requirements |

Do not silently default to Python or JavaScript. If the user has not chosen,
ask: “Use native GEE Code Editor JavaScript, Python/geemap, or both?” Keep the
selection algorithm and selected scene IDs identical across backends.

If candidate selection has already finished but the backend is still unknown,
pause at this same gate: acknowledge that the selection is ready, ask the
backend question, and do not emit a preview, selection report, or copy-paste
code until the user answers. Reuse the existing selected combination after the
choice; do not rerun Earth Engine unnecessarily.

Every successful or `incomplete_masked` run must return or save:

1. A preview appropriate to the chosen backend. Native JavaScript must include
   `Map.addLayer` and `print`; Python/geemap must create a viewable map object
   and, for a non-notebook run, save an HTML preview and preferably a PNG.
2. A selection report in Markdown or JSON containing required tile IDs, raw and
   incremental ROI coverage, selected scene IDs, UTC acquisition dates, local
   cloud and clear fractions, image footprint coverage, date span, final visible
   clear fraction, final masked gap, `overlapMode`, priority order, candidate
   count, and explicit `fallback`/`status`.
3. A copy-paste handoff script. It must import one exact scene per required
   tile, apply the dataset's cloud/shadow/snow/saturation mask and scale
   transform, assemble the scenes in the reported priority order, expose a
   `mosaic`/`composite` variable, and leave clipping/export as an explicit next
   step. For `priority_mosaic`, lower-priority scenes come first and the anchor
   scene is last because `mosaic()` uses the last unmasked pixel on top.

The handoff script is allowed to be short: it does not need to rerun candidate
search. It must not use an unfiltered date collection, `qualityMosaic`, a
cross-date `median`, or `unmask` with reflectance values. The report must state
that its dates and cloud fractions are the provenance of the selected coupled
combination, not newly optimized pixels.

For `no_solution`, return the diagnostics and failure status without fabricating
a complete handoff. A handoff is allowed only when it is explicitly labeled
`incomplete_masked` or another documented user-approved fallback. For
Sentinel-2, a handoff must preserve the joined
`COPERNICUS/S2_CLOUD_PROBABILITY/<scene-id>` mask (or explicitly document an
SCL-only fallback), not silently switch to a different cloud rule.

## Input Contract

Collect or state these parameters before writing code:

- `roi`: study-area geometry.
- `dataset`: default `COPERNICUS/S2_SR_HARMONIZED`; use Landsat Collection 2 Level-2 when its resolution or record is required.
- `start`, `end`: date window. Treat `end` as exclusive; support a month filter such as May-September inside a multi-year window.
- `targetDate` (optional): date used to break ties or limit temporal drift.
- `minCoverage`: required ROI coverage, usually `0.99` for tile geometry and a separate image-level coverage check.
- `baseCloudThreshold`: optional local cloud threshold for the anchor tile.
- `otherTileCloudThreshold`: local cloud threshold for other tiles, often looser (for example 0.20).
- `maxDateGapDays`: maximum allowed difference from the anchor acquisition; never widen it silently.
- `topN`: number of candidate scenes retained per tile before combination evaluation.
- `overlapMode`: default `priority_mosaic`; use `exclusive_tile` when strict tile ownership is required.
- `language`: required user choice: GEE Code Editor JavaScript, Python/geemap, or both. Ask before querying Earth Engine.
- `export`: preview only by default; create export tasks only when explicitly requested.

If the user asks for several years of the same season but wants a single output, say that the result is a global best seasonal mosaic from the whole window, not a representative image for every year.

## Core Workflow

### 1. Determine the minimum tile set

Use tile footprints and the ROI, not image cloud metadata, to determine geometric coverage.

1. Filter the dataset by `filterBounds(roi)` and identify distinct tile IDs (`MGRS_TILE` for Sentinel-2; path/row for Landsat).
2. Sort tiles by ROI intersection area.
3. Add tiles greedily until the union covers `minCoverage` of the ROI. Compute each tile's **incremental contribution** after subtracting the union of previously selected tiles; do not add overlapping raw areas.
4. Keep the ordered tile list. The first tile is the default anchor tile because it contributes the most unique ROI area.

Call this `minimum_geometry_tiles`. Track `minimum_usable_tiles` separately: a geometrically required tile with no acceptable scene is a quality failure, not permission to substitute a distant date without reporting it.

### 2. Build ROI-local candidate metrics

For every image in each required tile:

- Apply the dataset's cloud, cloud-shadow, cirrus, snow, saturation, and edge masks.
- Compute `footprintCoverage` over `roi ∩ image footprint`.
- Compute `localCloudFraction = cloudy pixels / covered pixels` within the tile's ROI portion. Never use the whole-scene cloud percentage as the final metric.
- Compute `localClearFraction`, acquisition timestamp, tile ID, scene ID, and distance to `targetDate` when supplied.
- Treat zero coverage, all-masked results, missing cloud-probability joins, and null reducer values explicitly.

Retain the best `topN` candidates per tile, but preserve enough alternatives for the anchor-date choice to affect the other tiles.

### 3. Evaluate coupled tile/date combinations

Do not permanently select the anchor scene before checking the other tiles.

1. Keep several candidate scenes for the anchor tile.
2. For each anchor candidate, filter other-tile candidates by local cloud threshold and `maxDateGapDays`.
3. Choose or enumerate matching candidates for every required tile. With few tiles, evaluate the Cartesian product of the top candidates; with many tiles, use a bounded beam/greedy search and state the approximation.
4. Assemble each candidate combination in the intended tile priority order and evaluate the **final visible ROI result**, so a lower tile's masked area that is hidden by a valid upper tile does not reduce the result.

Use hard constraints first, then an explainable lexicographic ranking:

1. required ROI coverage and per-tile quality thresholds;
2. highest final visible clear coverage;
3. lowest final masked/invalid gap fraction;
4. smallest tile-to-anchor date span;
5. closest distance to `targetDate` (if supplied);
6. lowest remaining local cloud fraction.

Expose a `temporal_first` policy when date coherence must outrank a small cloud improvement. Do not hide a weighted score behind an unexplained constant.

### 4. Assemble without implicit date mixing

Mask each selected image before mosaicking. In the default `priority_mosaic` mode, place the anchor tile above the next tile, and so on. Earth Engine's `mosaic()` uses the last unmasked image in collection order as the top image; order the collection accordingly and verify it in the generated code.

The lower-priority tile may fill an upper tile's masked gap, but it must never replace a valid upper-tile pixel. This preserves the user's tile priority while using overlap to reduce visible cloud.

Use `exclusive_tile` when every ROI location must belong to exactly one tile and no lower tile may fill another tile's cloud gap. In either mode, retain per-tile dates and visible contribution diagnostics.

### 5. Handle failure explicitly

If no combination satisfies the constraints, return one of these explicit states:

- `incomplete_masked`: best feasible combination with masked gaps;
- `relaxed_cloud`: thresholds were relaxed by a user-approved schedule;
- `expanded_window`: the date window was expanded only with explicit permission;
- `no_solution`: no scene combination is acceptable.

Never call `.first()` on an unchecked empty collection and never silently expand the date window.

## Output Contract

Return or print:

- selected tile IDs and one scene ID/date per tile;
- incremental and final ROI coverage;
- local cloud and clear fractions per tile;
- final visible clear and masked-gap fractions after priority mosaicking;
- `dateSpanDays`, `targetDateDistanceDays`, `fallback`, and the reason for any fallback;
- overlap mode and tile priority order.

Also deliver the backend-specific handoff:

### Native GEE Code Editor JavaScript

Return a runnable `.js` block with no `import geemap`, local file paths, or
Python-only authentication. It should contain exact image IDs such as
`ee.Image('LANDSAT/LC08/C02/T1_L2/<scene-id>')` or
`ee.Image('COPERNICUS/S2_SR_HARMONIZED/<scene-id>')`, a matching mask helper,
the selected images in priority order, `var mosaic = ...`, `Map.addLayer`, and
`print` diagnostics. Use a clear placeholder or the user's asset for `roi`.

### Python/geemap

Return the full selection/preview script plus a compact handoff block that
imports the same scene IDs, applies the same masks and scale factors, builds
the same mosaic, and exposes `mosaic` for subsequent analysis. Initialize with
the confirmed Cloud Project ID, save a viewable map, and do not start exports
unless explicitly requested.

### Report code requirements

The report's code must make overlap semantics visible. For example:

```javascript
var sceneLower = maskScene(ee.Image('DATASET/LOWER_SCENE_ID'));
var sceneAnchor = maskScene(ee.Image('DATASET/ANCHOR_SCENE_ID'));
var mosaic = ee.ImageCollection.fromImages([
  sceneLower,   // lower priority first
  sceneAnchor   // anchor last / top priority
]).mosaic();
var clipped = mosaic.clip(roi);
```

Replace the placeholders with the actual selected IDs and dataset-specific
mask; do not present this generic block as the final result.

Keep cloudy pixels masked. Do not `unmask()` with reflectance values unless the user explicitly requests a documented NoData value. For exports, specify `region`, `scale`, `crs`, `maxPixels`, and destination; do not start tasks by default. Cast export bands to compatible dtypes.

## References

- Read [selection-design.md](references/selection-design.md) for the set-cover, candidate-combination, scoring, overlap, and seasonal-window design.
- Read [sentinel2-javascript.md](references/sentinel2-javascript.md) for a Code Editor pattern using Sentinel-2 SR Harmonized and cloud probability.
- Read [python-geemap.md](references/python-geemap.md) for the equivalent Python API/geemap pattern.
- Read [landsat.md](references/landsat.md) when using Landsat Collection 2 Level-2.

## Common Mistakes

- Ranking by `CLOUDY_PIXEL_PERCENTAGE` or `CLOUD_COVER` without an ROI-local metric.
- Summing overlapping tile areas instead of incremental coverage.
- Choosing one lowest-cloud anchor scene and never evaluating alternative anchor dates.
- Using `qualityMosaic`, cross-date `median`, or a date-unaware `mosaic` over the entire collection.
- Putting images in the wrong collection order and accidentally giving the wrong tile priority.
- Computing cloud fraction with a denominator that includes pixels outside the tile footprint.
- Treating a five-year seasonal search as an interannual representative product without saying so.
- Filling every masked pixel from another date and calling the result a single-date image.
- Asking for or returning only a full selector when the user needs a small downstream script.
- Returning raw scene IDs without the same mask, scale/offset, or priority order used during selection.
- Reporting scene dates without UTC provenance, final visible metrics, or fallback/status.
- Letting a copy-paste handoff silently bypass the selected date-gap and one-scene-per-tile constraints.
