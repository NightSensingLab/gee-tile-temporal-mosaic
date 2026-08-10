# Python/geemap Pattern

## Contents

- [Core Python pattern](#core-python-pattern)
- [Python-specific guardrails](#python-specific-guardrails)

Use the same parameter names and selection semantics as the JavaScript reference. The following pattern creates ROI-local Sentinel-2 candidates; the tile set and coupled scene search remain project-specific.

## Core Python pattern

```python
import ee
import geemap

PROJECT_ID = "PROJECT_ID"
ee.Initialize(project=PROJECT_ID)

roi = ee.FeatureCollection("users/PROJECT/ROI").geometry()
start = "2019-05-01"
end = "2024-10-01"  # end is exclusive
metric_scale = 20
cloud_probability_threshold = 40
min_tile_coverage = 0.01

s2 = (
    ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")
    .filterBounds(roi)
    .filterDate(start, end)
)
cloud_probability = (
    ee.ImageCollection("COPERNICUS/S2_CLOUD_PROBABILITY")
    .filterBounds(roi)
    .filterDate(start, end)
)

joined = ee.ImageCollection(
    ee.Join.saveFirst("cloud_probability").apply(
        primary=s2,
        secondary=cloud_probability,
        condition=ee.Filter.equals(
            leftField="system:index", rightField="system:index"
        ),
    )
).filter(ee.Filter.notNull(["cloud_probability"]))


def add_metrics(image):
    probability = ee.Image(image.get("cloud_probability")).select("probability")
    scl = image.select("SCL")
    scl_clear = (
        scl.neq(3)
        .And(scl.neq(8))
        .And(scl.neq(9))
        .And(scl.neq(10))
        .And(scl.neq(11))
    )
    edge = image.select("B8A").mask().And(image.select("B9").mask())
    clear = probability.lt(cloud_probability_threshold).And(scl_clear).And(edge)

    covered = (
        ee.Image.constant(1)
        .clip(image.geometry())
        .clip(roi)
        .unmask(0)
        .rename("covered")
    )
    clear01 = clear.unmask(0).rename("clear")
    cloudy01 = clear.Not().unmask(0).rename("cloudy")
    stats = ee.Image.cat(
        [
            covered,
            covered.multiply(clear01).rename("clear_in_roi"),
            covered.multiply(cloudy01).rename("cloudy_in_roi"),
        ]
    ).reduceRegion(
        reducer=ee.Reducer.mean(),
        geometry=roi,
        scale=metric_scale,
        maxPixels=10**9,
        bestEffort=True,
        tileScale=4,
    )

    coverage = ee.Number(stats.get("covered"))
    safe_coverage = coverage.max(1e-6)
    return image.updateMask(clear).set(
        {
            "tile_id": image.get("MGRS_TILE"),
            "scene_id": image.get("system:index"),
            "acquisition_millis": image.get("system:time_start"),
            "roi_coverage": coverage,
            "local_clear_fraction": ee.Number(stats.get("clear_in_roi")).divide(
                safe_coverage
            ),
            "local_cloud_fraction": ee.Number(stats.get("cloudy_in_roi")).divide(
                safe_coverage
            ),
            "coverage_ok": coverage.gte(min_tile_coverage),
        }
    )


measured = joined.map(add_metrics)
tile_candidates = (
    measured.filter(ee.Filter.eq("tile_id", "T50SNA"))
    .filter(ee.Filter.gte("roi_coverage", min_tile_coverage))
    .sort("local_cloud_fraction")
)
print(tile_candidates.limit(10).getInfo())

# After the coupled search chooses one masked image per tile, order lower
# priority images first and the anchor last. Earth Engine's mosaic uses the
# last unmasked image on top.
# selected = ee.ImageCollection.fromImages([scene_c, scene_b, scene_a])
# mosaic = selected.mosaic().clip(roi)

# Preview with geemap; exports remain opt-in.
# Map = geemap.Map()
# Map.centerObject(roi, 10)
# Map.addLayer(mosaic, {"bands": ["B4", "B3", "B2"], "min": 0, "max": 3000}, "mosaic")
# Map
```

## Python-specific guardrails

- Initialize with an explicit Cloud Project ID before making Earth Engine calls.
- Keep reducers and candidate properties server-side; do not loop over large collections with `getInfo()`.
- For a large ROI, use `tileScale`, `bestEffort`, or a controlled tile-by-tile diagnostic pass.
- Use `geemap` only for visualization/download convenience; the selection semantics remain Earth Engine server-side.
- Export only after confirming the resolved tile dates, coverage, date span, CRS, scale, and destination.
