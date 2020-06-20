# pdal-pipelines

Some pipelines used for routine point cloud processing with the **[point data abstraction library](https://pdal.io)** (PDAL).

## Metadata production

### [geoJSONbounds.json](./geoJSONbounds.json)

A simple pipeline to generate point cloud boundaries in WGS84 lat/lon (EPSG:4326). It uses a [reprojection filter](https://pdal.io/stages/filters.reprojection.html) to express points in WGS84 lat/lon, then the [hexbin filter](https://pdal.io/stages/filters.hexbin.html) to generate a boundary.

In this case we don't write out any points - so we need to run like:

`pdal pipeline geoJSONbounds.json --metadata > pdal-bounds.json`

....which needs to be read (for example using `jq`) to get a geoJSON geometry:

`cat pdal-bounds.json | jq .stages["filters.hexbin"].boundary_json' > pdal-bounds.geojson`

## Point processing / modification

### [elm-csf-hag.json](./processing/elm-csf-hag.json)

This pipeline is a standard workflow used by [Spatialised](https://spatialised.net) for point clouds derived from structure-from-motion photogrammetry (usually flown using small RPA or drones). In general the cloth sampling filter is able to produce the best 'ground' labels out of methods available in PDAL for structure-from-motion derived point clouds.

It sets all classes in an input las file to `0`, then runs an [extended local minima filter](https://pdal.io/stages/filters.elm.html) to flag low points as noise (class `7`). It then runs a [cloth simulation filter](https://pdal.io/stages/filters.csf.html) to identify and label `ground` points (ASPRS class `2`). Finally, it uses a [local delaunay triangulation filter](https://pdal.io/stages/filters.hag_delaunay.html) to determine 'height above ground' for non-ground points.

## [sort-points.json](./sort-points.json)

Point clouds from 3D laser scanners often get really memory intensive for further processing (conversion to EPT is a common mind bender).


## Point filtering

### [vegabove3m-bounds.json](./filtering/vegabove3m-bounds.json)

This pipeline is used to filter a point cloud, keeping only points which are labelled as vegetation and have a 'normalised height' (height above ground) of 3m or greater.

In this pipeline we rely on input data being labelled with `vegetation` classes, usually ASPRS classes `3`, `4`, and `5`. We also rely on data having a `HeightaboveGround` dimension. It uses a [range filter](https://pdal.io/stages/filters.range.html) to exclude any other points from subsequent analysis, and further removes any points with `HeightAboveGround` lower than 3 vertical units (usually metres).

Once that's done, we use a [hexbin filter](https://pdal.io/stages/filters.hexbin.html) to collect bounding polygons around whatever points we have left. Finally we write the points out as a LASzip compressed LAS file. This one needs a small modification to its run command in order to capture the output of `filters.hexbin`, piping `metadata` output to a JSON file:

`pdal pipeline vegabove3m.json --metadata > veg-bounds-pdaldata.json`

....which needs to be read (for example using `jq`) to get a geoJSON geometry:

`cat veg-bounds-pdaldata.json | jq .stages["filters.hexbin"].boundary_json' > bounds.geojson`

**Note:** the resulting GeoJSON coordinates are in the same coordinate system as the input LAS/LAZ file. Strictly GeoJSON is always expressed in WGS84 lat/lon (EPSG:4326), so you may choose to transform the coordinates at some later point.


## Raster production

### [dtm-fullres.json](./raster/dtm-fullres.json)

This is a very straightforward pipeline which generates a digital terrain model from a point cloud which has ground-labelled (ASPRS class `2`) points and is not decimated in any way. It is generally run after something like `elm-csf-hag.json` to generate a raster terrain model.

It ingests a point cloud, and uses a [range filter](https://pdal.io/stages/filters.range.html) to retain only points labelled with `Classification` of `2`. It then creates [raster output using GDAL](https://pdal.io/stages/writers.gdal.html), in this case a 6-band geoTIFF containing:

| band | value |
|-----|------- |
| 1 | minimum Z value within the raster cell |
| 2 | maximum Z value within the raster cell|
| 3 | mean Z within cell |
| 4 | cell Z value derived by inverse distance weighting using surrounding cells |
| 5 | count of points within the cell |
| 6 | standard deviation of Z values within the cell |

The output tiff is 32 bit by default. Run without modifiers this pipeline produces DTMs with 2m x 2m pixel resolution. It also attempts to fill data holes by interpolating values from up to 10 cells away from the hole.

### [dtm-dartsample.json](./raster/dtm-dartsample.json)

Like [dtm-fullres.json](./dtm-fullres.json), this pipeline results in a geoTIFF dtm, using only ground-labelled points. In this case we reduce the data density using a [poisson dart throwing method](https://pdal.io/stages/filters.sample.html). This helps to overcome varying point densities when handling data with, for example, many overlapping flight lines which significantly skew point spatial density.

In this pipeline, the point sampling method is set to ensure that points are no less than 1.5 m (in the XY plane) apart.

After data reduction, a geotiff is produced - this time using only a single band, `idw` (band 4 in the table above).


### [vegabove3m-raster.json](./raster/vegabove3m-raster.json)

This pipeline generates a 2 metre resolution raster using points classified as vegetation and with a height above ground of 3 m or taller.  
