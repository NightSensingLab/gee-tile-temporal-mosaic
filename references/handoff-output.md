# Handoff and Report Output

The expensive part of a tile-temporal mosaic is selecting the coupled scene
combination. The handoff is intentionally small: it lets a researcher continue
in the Code Editor or in Python without rerunning the search.

## Required report fields

Report one row per selected tile with:

```text
tile_id
scene_id
acquisition_date_utc
roi_coverage
local_clear_fraction
local_cloud_fraction
```

Also report `minimum_geometry_tiles`, raw/incremental tile coverage, the anchor
tile, date span, final visible clear fraction, final masked gap, candidate count,
`overlap_mode`, priority order, and `fallback`/`status`. A seasonal multi-year
result must say whether it is one global best window result or one result per
year.

## Native Code Editor handoff

The generated JavaScript must be self-contained apart from the user's `roi`
asset or geometry. Use the exact selected scene asset IDs and the same dataset
mask used during selection:

```javascript
// Supply the study-area geometry in the Code Editor.
var roi = ee.FeatureCollection('users/PROJECT/ROI').geometry();

// Use the dataset-specific helper from references/landsat.md or
// references/sentinel2-javascript.md. It must preserve the image mask.
function maskScene(image) {
  return image.updateMask(/* same clear mask used during selection */);
}

var sceneLower = maskScene(
  ee.Image('DATASET/EXACT_LOWER_SCENE_ID')
);
var sceneAnchor = maskScene(
  ee.Image('DATASET/EXACT_ANCHOR_SCENE_ID')
);

// Last image is top priority in ImageCollection.mosaic().
var mosaic = ee.ImageCollection.fromImages([
  sceneLower,
  sceneAnchor
]).mosaic();
var clipped = mosaic.clip(roi);

print('Selected scenes', {
  lower: 'EXACT_LOWER_SCENE_ID',
  anchor: 'EXACT_ANCHOR_SCENE_ID'
});
Map.centerObject(roi, 10);
Map.addLayer(clipped, {bands: ['B5', 'B4', 'B3'], min: 0.02, max: 0.35}, 'mosaic');
```

The placeholder mask is not a final script. Replace it with the full
dataset-specific mask and scale/offset transformation, and replace every
placeholder scene ID before presenting the code as copy-paste ready. For
Landsat Collection 2 Level-2, use the `SR_B*` scale/offset and `QA_PIXEL` /
`QA_RADSAT` rules. For Sentinel-2, use the joined cloud-probability image,
SCL, and edge masks.

The output must contain one selected scene per required tile, not a new
`filterDate()` collection. Keep lower-priority scenes first and the anchor last.

## Python/geemap handoff

The Python handoff uses the same exact scene IDs and order:

```python
scene_lower = mask_scene(ee.Image("DATASET/EXACT_LOWER_SCENE_ID"))
scene_anchor = mask_scene(ee.Image("DATASET/EXACT_ANCHOR_SCENE_ID"))
mosaic = ee.ImageCollection.fromImages([scene_lower, scene_anchor]).mosaic()
clipped = mosaic.clip(roi)
```

The full Python run initializes the confirmed Cloud Project ID and saves a
viewable geemap map. The compact handoff must expose `mosaic` and must not
start an export task without an explicit user request.

## Integrity checks

Before returning the handoff, verify that:

- every scene ID exists in the requested collection;
- the number of imported scenes equals the number of required tiles;
- dates and local cloud metrics match the selected combination report;
- the collection order matches `overlap_mode` and the documented priority;
- cloud pixels remain masked and no reflectance `unmask` was introduced;
- the reported date-gap constraint is still true for every non-anchor tile.

