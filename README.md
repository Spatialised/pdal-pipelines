# pdal-pipelines
Pipelines for routine tasks using the point data abstraction library (PDAL)

## elm-csf-hag.json

This pipeline sets all classes in an input las file to `0`, then runs an extended local minima filter to flag low points as noise (class `7`). It then runs a cloth simulation filter to identify and label `ground` points (ASPRS class `2`). Finally, it uses a local delaunay triangulation filter to determine 'height above ground' for non-ground points.
