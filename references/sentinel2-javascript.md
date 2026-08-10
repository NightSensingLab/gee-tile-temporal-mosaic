# Sentinel-2 JavaScript Pattern

This is a Code Editor pattern for producing ROI-local metrics. It intentionally stops before a project-specific tile set and candidate table are chosen; use the selection rules in [selection-design.md](selection-design.md) to select one scene per tile before assembling the final mosaic.

```javascript
// Replace these placeholders in the Code Editor.
var roi = ee.FeatureCollection('users/PROJECT/ROI').geometry();
var start = '2019-05-01';
var end = '2024-10-01'; // end is exclusive
var metricScale = 20;
var cloudProbabilityThreshold = 40;
var minTileCoverage = 0.01;

var s2 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(roi)
  .filterDate(start, end);

var cloudProbability = ee.ImageCollection('COPERNICUS/S2_CLOUD_PROBABILITY')
  .filterBounds(roi)
  .filterDate(start, end);

var joined = ee.ImageCollection(ee.Join.saveFirst('cloud_probability').apply({
  primary: s2,
  secondary: cloudProbability,
  condition: ee.Filter.equals({
    leftField: 'system:index',
    rightField: 'system:index'
  })
})).filter(ee.Filter.notNull(['cloud_probability']));

function addClearMaskAndMetrics(image) {
  var probability = ee.Image(image.get('cloud_probability'))
    .select('probability');
  var scl = image.select('SCL');

  // Keep these classes configurable for snow/bright-surface studies.
  var sclClear = scl.neq(3)   // cloud shadow
    .and(scl.neq(8))           // medium probability cloud
    .and(scl.neq(9))           // high probability cloud
    .and(scl.neq(10))          // cirrus
    .and(scl.neq(11));         // snow/ice
  var edge = image.select('B8A').mask().and(image.select('B9').mask());
  var clear = probability.lt(cloudProbabilityThreshold)
    .and(sclClear)
    .and(edge);

  // `covered` is the ROI portion inside the image footprint. Unmasking these
  // diagnostic 0/1 images makes the reducer denominator the whole ROI.
  var covered = ee.Image.constant(1)
    .clip(image.geometry())
    .clip(roi)
    .unmask(0)
    .rename('covered');
  var clear01 = clear.unmask(0).rename('clear');
  var cloudy01 = clear.not().unmask(0).rename('cloudy');

  var stats = ee.Image.cat([
    covered,
    covered.multiply(clear01).rename('clear_in_roi'),
    covered.multiply(cloudy01).rename('cloudy_in_roi')
  ]).reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: roi,
    scale: metricScale,
    maxPixels: 1e9,
    bestEffort: true,
    tileScale: 4
  });

  var coverage = ee.Number(stats.get('covered'));
  var clearInRoi = ee.Number(stats.get('clear_in_roi'));
  var cloudyInRoi = ee.Number(stats.get('cloudy_in_roi'));
  var safeCoverage = coverage.max(1e-6);

  return image
    .updateMask(clear)
    .set({
      tile_id: image.get('MGRS_TILE'),
      scene_id: image.get('system:index'),
      acquisition_millis: image.get('system:time_start'),
      roi_coverage: coverage,
      local_clear_fraction: clearInRoi.divide(safeCoverage),
      local_cloud_fraction: cloudyInRoi.divide(safeCoverage),
      coverage_ok: coverage.gte(minTileCoverage)
    });
}

var measured = joined.map(addClearMaskAndMetrics);

// Example: inspect candidates for one already chosen tile. Replace TILE_ID
// and use the candidate properties to build the coupled tile/date search.
var TILE_ID = 'T50SNA';
var tileCandidates = measured
  .filter(ee.Filter.eq('tile_id', TILE_ID))
  .filter(ee.Filter.gte('roi_coverage', minTileCoverage))
  .sort('local_cloud_fraction');

print('Top local candidates', tileCandidates.limit(10));

// After selecting one image per required tile, place lower-priority images
// first and the anchor image last: mosaic() uses the last unmasked pixel on
// top. All images must already be cloud-masked.
// var priorityMosaic = ee.ImageCollection.fromImages([sceneC, sceneB, sceneA])
//   .mosaic()
//   .clip(roi);

// Preview only. Create an export task only after confirming parameters.
// Export.image.toDrive({
//   image: priorityMosaic.toFloat(),
//   description: 'tile_temporal_mosaic',
//   region: roi,
//   scale: 10,
//   maxPixels: 1e13,
//   fileFormat: 'GeoTIFF'
// });
```

## Adaptation rules

- Use a month filter after `filterDate` for a multi-year seasonal window, but preserve the year in every candidate property.
- Compute geometry tile contributions before image quality metrics. The candidate's local cloud denominator must be the ROI portion covered by that image.
- The example's `clear.not()` treats missing probability pixels as cloudy after `unmask(0)`. For a different missing-data policy, make it explicit and record it.
- `mosaic()` only gives date-coherent output when the collection contains the one selected image for each required tile. Never pass the unfiltered date collection to `mosaic()`.
