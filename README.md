# pdal-pipelines
Pipelines for routine tasks using the point data abstraction library (PDAL).

## elm-csf-hag.json

This pipeline sets all classes in an input las file to `0`, then runs an extended local minima filter to flag low points as noise (class `7`). It then runs a cloth simulation filter to identify and label `ground` points (ASPRS class `2`). Finally, it uses a local delaunay triangulation filter to determine 'height above ground' for non-ground points.

It is a standard workflow used by Spatialised for point clouds derived from structure-from-motion photogrammetry (usually flown using small RPA or drones)

## dtm-fullres.json

This is a very straightforward pipeline which generates a digital terrain model from a point cloud which has ground-labelled (ASPRS class `2`) points - and is generally run after `elm-csf-hag.json`. It ingests a point cloud, and using only points labelled with `Classification` of `2`, creates a raster output. This version of the filter results in a 6-band geotiff containing:

| band | value |
|-----|------- |
| 1 | minimum Z value |
| 2 | maximum Z value |
| 3 | mean Z within cell |
| 4 | cell Z value derived by inverse distance weighting using surrounding cells |
| 5 | count of points within the cell |
| 6 | standard deviation of Z values within the cell |

The output tiff is 32 bit by default. Run without modifiers this pipeline produces DTMs with 2m resolution. It also attempts to fill data holes by interpolating values from up to 10 cells away from the hole.
