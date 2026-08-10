# Selection Design

## Contents

- [Geometry stage](#1-geometry-stage)
- [Candidate table](#2-candidate-table)
- [Coupled search](#3-coupled-search)
- [Overlap semantics](#4-overlap-semantics)
- [Fallback ladder](#5-fallback-ladder)
- [Recommended diagnostics](#6-recommended-diagnostics)

## 1. Geometry stage

The geometric stage chooses the fewest tile IDs capable of covering the ROI. For an ordered tile list `T`, define:

```text
incrementalContribution(t) = area((t ∩ ROI) - union(previousTiles ∩ ROI)) / area(ROI)
```

Add the tile with the largest incremental contribution until the union reaches `minCoverage`. This is a greedy set-cover approximation. It is normally sufficient because Sentinel-2 and Landsat ROI intersections contain few candidate tiles. If the ROI spans many tiles, compare the greedy result with a bounded exact search over the top candidates.

Do not use raw intersection percentages when tile footprints overlap. Keep both the raw intersection and incremental contribution for diagnostics.

## 2. Candidate table

Build one row per image candidate:

```text
tile_id
scene_id
acquisition_time
footprint_coverage
local_cloud_fraction
local_clear_fraction
local_shadow_fraction
target_date_distance_days
```

The denominator for local cloud is the valid footprint inside the ROI portion of that tile. A cloud mask should be conservative: cloud probability plus scene classification for Sentinel-2; QA_PIXEL and saturation flags for Landsat. A zero denominator is a rejected candidate, not zero cloud.

For a multi-year month window, preserve the full timestamp and year. A single global result can legitimately come from any year, but the output must state that interpretation.

## 3. Coupled search

The anchor tile is selected by geometric contribution, but its scene should not be fixed by one greedy `min(cloud)` call. Keep `topN` candidates and evaluate combinations:

```text
best = none
for anchor in anchorCandidates:
    matches = candidates for each other tile
        where localCloud <= tileThreshold
        and abs(date - anchor.date) <= maxDateGapDays
    for combination in boundedCombinations(anchor, matches):
        result = priorityMosaic(combination)
        metrics = finalVisibleMetrics(result, ROI)
        rank by hard constraints, then:
            finalVisibleClearFraction desc
            finalMaskedGapFraction asc
            dateSpanDays asc
            targetDateDistanceDays asc
            totalLocalCloud asc
```

If the product prioritizes temporal coherence, switch the second and third ranking terms or enforce a smaller `maxDateGapDays`. If the number of tiles or candidates makes the Cartesian product too large, retain the best `topN` per tile and use beam search; report that the search is bounded.

## 4. Overlap semantics

`priority_mosaic` is the recommended default for the user's objective:

```text
anchor tile > second contribution tile > third contribution tile
```

Cloudy pixels must be masked before composition. A lower tile can appear only where every higher-priority image is masked or absent. Thus a cloudy lower tile hidden by a valid upper tile does not reduce final visible quality. A lower tile filling an upper tile's masked gap is a deliberate, diagnosable date difference. Because masked cloud pixels are not visible in the final image, report final clear coverage and masked-gap fraction; use the pre-mask local cloud fraction for candidate quality.

Use `exclusive_tile` when the analysis cannot tolerate any cross-tile gap filling. Assign ROI locations to tiles by a deterministic ownership mask based on the priority order, then use only the selected scene for each owner.

Do not use `qualityMosaic` over dates: it chooses a source independently per pixel and destroys the single-acquisition-per-tile guarantee.

## 5. Fallback ladder

Keep fallback explicit and ordered:

1. exact user window, configured thresholds, configured date gap;
2. user-approved cloud-threshold relaxation;
3. user-approved date-gap relaxation;
4. user-approved date-window expansion;
5. incomplete masked output or `no_solution`.

Adding a fourth tile is a geometry/availability decision and must be reported separately from relaxing a cloud threshold. Never silently turn a one-scene-per-tile product into a cross-date per-pixel composite.

## 6. Recommended diagnostics

Attach image properties or a companion FeatureCollection containing:

- `tile_ids`, `tile_dates`, and `scene_ids`;
- raw and incremental tile coverage;
- local cloud and shadow fractions;
- visible contribution by tile after priority layering;
- `final_clear_fraction`, `final_masked_gap_fraction`, `date_span_days`;
- `selection_policy`, `overlap_mode`, and fallback fields.

The final response should include a compact handoff using the selected scene IDs,
not only the diagnostic table. The handoff must reapply the same mask and scale
transform, import exactly one scene per required tile, preserve the reported
priority order, and expose `mosaic`/`composite` for downstream clipping and
analysis. See [handoff-output.md](handoff-output.md).
