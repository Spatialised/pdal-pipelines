# pdal-pipelines

Some pipelines used for routine point cloud processing with the **[point data abstraction library](https://pdal.io)** (PDAL).

## [elm-csf-hag.json](./elm-csf-hag.json)

This pipeline sets all classes in an input las file to `0`, then runs an [extended local minima filter](https://pdal.io/stages/filters.elm.html) to flag low points as noise (class `7`). It then runs a [cloth simulation filter](https://pdal.io/stages/filters.csf.html) to identify and label `ground` points (ASPRS class `2`). Finally, it uses a [local delaunay triangulation filter](https://pdal.io/stages/filters.hag_delaunay.html) to determine 'height above ground' for non-ground points.

It is a standard workflow used by Spatialised for point clouds derived from structure-from-motion photogrammetry (usually flown using small RPA or drones). In general the cloth sampling filter is able to produce the best 'ground' labels out of methods available in PDAL for structure-from-motion derived point clouds.

## [dtm-fullres.json](./dtm-fullres.json)

This is a very straightforward pipeline which generates a digital terrain model from a point cloud which has ground-labelled (ASPRS class `2`) points and is not decimated in any way. It is generally run after `elm-csf-hag.json` to generate a raster terrain model.

It ingests a point cloud, and uses a [range filter](https://pdal.io/stages/filters.range.html) to retain only points labelled with `Classification` of `2`. It then creates [raster output using GDAL](https://pdal.io/stages/writers.gdal.html), in this case a 6-band geoTIFF containing:

| band | value |
|-----|------- |
| 1 | minimum Z value |
| 2 | maximum Z value |
| 3 | mean Z within cell |
| 4 | cell Z value derived by inverse distance weighting using surrounding cells |
| 5 | count of points within the cell |
| 6 | standard deviation of Z values within the cell |

The output tiff is 32 bit by default. Run without modifiers this pipeline produces DTMs with 2m resolution. It also attempts to fill data holes by interpolating values from up to 10 cells away from the hole.

## [dtm-dartsample.json](./dtm-dartsample.json)

Like [dtm-fullres.json](./dtm-fullres.json), this pipeline results in a geoTIFF dtm, using only ground-labelled points. Before assembling the raster output, points are filtered using a [poisson dart throwing method](https://pdal.io/stages/filters.sample.html). In this pipeline, the point sampling method is set to ensure that points are no less than 1.5 m (in the XY plane) apart.

After data reduction, a geotiff is produced - this time using only a single band, `idw` (band 4 in the table above).

This pipeline is useful when handling data with widely varying density (eg overlapping flight lines).
