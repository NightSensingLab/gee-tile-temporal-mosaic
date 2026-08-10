# Three-Tile Seasonal Request

## Request

```text
Use $gee-tile-temporal-mosaic. My ROI spans three Sentinel-2 MGRS tiles.
Search 2019-2023 May-September. Require anchor local cloud <= 3%, other-tile
local cloud <= 20%, and a maximum five-day gap from the anchor. Keep one scene
per tile, evaluate several anchor candidates, use priority_mosaic, and output
GEE Code Editor JavaScript. Print tile dates, local cloud fractions, final
clear coverage, masked-gap fraction, and fallback state. Do not use
qualityMosaic or cross-date median.
```

Before running, ask whether the user wants native Code Editor JavaScript,
Python/geemap, or both. The selected backend must determine the primary script;
the response must also include a copy-paste import/mask/mosaic handoff.

## Expected decisions

1. Treat the result as one global best seasonal mosaic across the full five-year
   window, not one output per year.
2. Determine the minimum geometric tile set using incremental ROI coverage.
3. Keep several anchor scenes; do not choose the first lowest-cloud scene before
   checking the other tile dates.
4. Filter other-tile candidates by local cloud fraction and the five-day anchor
   gap, then evaluate complete tile/date combinations.
5. Assemble selected scenes in `[smallest contribution, ..., anchor]` collection
   order so the anchor is on top in `mosaic()`.

## Expected diagnostics

The response should print or attach:

- required tile IDs, raw and incremental coverage;
- one scene ID and acquisition date per tile;
- local cloud/clear/coverage metrics per tile;
- final clear coverage and masked-gap fraction;
- date span, selection policy, overlap mode, and fallback state.
- a copy-paste script importing exactly one selected scene per tile, applying the
  same mask, and exposing `mosaic` for later clipping and analysis;
- a preview artifact appropriate to the selected backend.

The response should explicitly state that it did not use `qualityMosaic`,
cross-date median, or implicit per-pixel date selection.
