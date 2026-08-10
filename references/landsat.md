# Landsat Collection 2 Reference

## Contents

- [Collections](#collections)
- [QA mask](#qa-mask)
- [Sensor mixing](#sensor-mixing)

Use this reference when the task needs Landsat Collection 2 Level-2 rather than Sentinel-2. Keep the tile-level selection and overlap rules from the main skill unchanged.

## Collections

- Landsat 8: `LANDSAT/LC08/C02/T1_L2`
- Landsat 9: `LANDSAT/LC09/C02/T1_L2`
- Use `LANDSAT/LC08/C02/T1_L2` and `LANDSAT/LC09/C02/T1_L2` together only after deciding how cross-sensor calibration will be handled.

Use `filterBounds(roi)`, `filterDate(start, end)`, and WRS-2 `WRS_PATH`/`WRS_ROW` as tile identifiers. `CLOUD_COVER` is a pre-filter only; calculate cloud fraction inside the ROI portion of each path/row scene.

## QA mask

For Collection 2 Level-2, use `QA_PIXEL` and `QA_RADSAT`. A conservative clear mask excludes dilated cloud, cirrus, cloud, cloud shadow, and snow:

```javascript
function landsatClear(image) {
  var qa = image.select('QA_PIXEL');
  var clear = qa.bitwiseAnd(1 << 1).eq(0)  // dilated cloud
    .and(qa.bitwiseAnd(1 << 2).eq(0))      // cirrus
    .and(qa.bitwiseAnd(1 << 3).eq(0))      // cloud
    .and(qa.bitwiseAnd(1 << 4).eq(0))      // cloud shadow
    .and(qa.bitwiseAnd(1 << 5).eq(0));     // snow
  var saturated = image.select('QA_RADSAT').eq(0);
  return clear.and(saturated);
}
```

Apply Collection 2 scale and offset before reflectance analysis:

```javascript
function scaleLandsat(image) {
  var optical = image.select('SR_B.*').multiply(0.0000275).add(-0.2);
  var thermal = image.select('ST_B.*').multiply(0.00341802).add(149.0);
  return image.addBands(optical, null, true)
    .addBands(thermal, null, true);
}
```

When computing ROI-local metrics, use the same `covered`, `clear`, and `reduceRegion` pattern as the Sentinel-2 reference. Keep the WRS path/row scene date and mask each selected image before the fixed-priority mosaic.

## Sensor mixing

Do not combine Landsat 8 and 9 with Sentinel-2 bands in one product without explicit band mapping, scale handling, and cross-sensor normalization. Report the sensor and processing collection for every selected tile.
