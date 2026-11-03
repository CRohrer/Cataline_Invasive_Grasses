# Catalina Island Invasive Species Mapping and Prediction

A geospatial remote-sensing and machine-learning workflow for identifying potential invasive plant occurrences on Santa Catalina Island, California, beyond areas already surveyed in the field. The project combines high-resolution aerial imagery, LiDAR-derived vegetation structure, terrain variables, and field-derived training polygons to explore supervised classification and unsupervised clustering.

## Project objectives

- Prepare and align imagery, elevation, canopy-height, and field-survey datasets in a common spatial reference system.
- Derive spectral, vegetation, and terrain predictors for classification.
- Use **Classification and Regression Trees (CART)** to distinguish a target invasive species from background samples.
- Explore **K-means and MiniBatchKMeans** clustering to identify spectral and structural patterns without species labels.
- Generate georeferenced classification rasters for further GIS review and validation.

## Data sources

| Dataset | Role in analysis | Source noted in notebook |
| --- | --- | --- |
| LARIAC7 four-inch orthorectified imagery (2023) | Blue, green, red, and near-infrared imagery; NDVI and HSV predictors | CSULB Geography Department / [LA County LARIAC](https://lariac-lacounty.hub.arcgis.com/pages/lariac7-documents-data) |
| LiDAR point clouds (including 2025 data) | Normalized point elevations and canopy-height models | [USGS National Map Downloader](https://apps.nationalmap.gov/downloader/) |
| DEM / digital terrain model | Terrain elevation, slope, and aspect | USGS National Map; DTM mosaic prepared in ArcGIS Pro |
| Vegetation, background, and management-unit shapefiles | Training samples, spatial clipping, and study-area boundaries | Client-provided data |

**Data availability:** The imagery and client-provided shapefiles are not bundled with this notebook. Access and redistribution may be restricted. Source URLs are provided for reference, not as a guarantee that the exact inputs are publicly downloadable.

## Processing workflow

1. **Coordinate systems and preprocessing:** Reproject vector and raster inputs to the working CRS (**EPSG:6424**), normalize imagery, and clip data to relevant Catalina Island management areas.
2. **Spectral features:** Extract RGB and NIR bands, calculate **Normalized Difference Vegetation Index (NDVI)**, and derive HSV representations of RGB and NIR-red-green (NRG) imagery.
3. **LiDAR and terrain:** Process point clouds, derive and mosaic canopy-height models (CHMs), and prepare DTM-based slope and aspect rasters.
4. **Raster alignment:** Align predictors to common spatial grids, inspect resolution and extents, and create multiband GeoTIFF feature stacks.
5. **Unsupervised classification:** Apply K-means or MiniBatchKMeans to valid pixels and export cluster-label rasters.
6. **Supervised classification:** Sample pixels within target-species and background polygons, train a binary CART decision tree, and predict classes (and optionally probabilities) across raster stacks.
7. **Evaluation:** Report held-out sample accuracy, precision, recall, F1-score, confusion matrix, and ROC AUC.

### Feature stacks

The notebook contains **both 15-band and 16-band processing paths**. The 15-band CART prediction function documents this order:

| Band | Feature | Band | Feature |
| --- | --- | --- | --- |
| 1 | Blue | 9 | NRG hue |
| 2 | Green | 10 | NRG saturation |
| 3 | Red | 11 | NRG value |
| 4 | NIR | 12 | DTM elevation |
| 5 | NDVI | 13 | Slope |
| 6 | RGB hue | 14 | Aspect |
| 7 | RGB saturation | 15 | 2025 canopy height |
| 8 | RGB value | | |

Other sections build or consume **16-band stacks**, including an additional canopy-height input. Confirm the band count and exact order for each processing function before reusing a stack: the notebook is an evolving workflow, not a single standardized pipeline.

## Methods

### K-means clustering

The notebook includes conventional `KMeans` and `MiniBatchKMeans` approaches. MiniBatchKMeans is used to help handle the large number of high-resolution raster pixels. Cluster IDs represent groups of similar pixel features; **they are not automatically invasive-species labels** and require interpretation or validation.

### CART classification

A `DecisionTreeClassifier` is trained using pixels sampled from target-species and background polygons. In the recorded example, samples are capped at **50,000 per class**, split **70% training / 30% testing**, stratified by class, and modeled with `max_depth=8` and `random_state=42`.

The notebook includes an example using *F. vulgare* as the target species and other species/background polygons as the negative class. Other species datasets are also referenced in the preprocessing workflow.

## Recorded results

The saved notebook output reports the following **binary CART test-set** metrics:

| Metric | Result |
| --- | ---: |
| Accuracy | **82.4%** |
| ROC AUC | **0.8996** |
| Target-class precision | **0.78** |
| Target-class recall | **0.90** |
| Target-class F1-score | **0.84** |
| Test samples | **30,000** |

Confusion matrix (rows: actual background/target; columns: predicted background/target):

```text
[[11233,  3767],
 [ 1513, 13487]]
```

These are **held-out pixel-sample metrics from the notebook**, not evidence of independently validated island-wide mapping accuracy. Spatial dependence, training-polygon representativeness, and generalization to unsurveyed areas need further assessment. The notebook does not establish a comparable quantitative accuracy result for the unsupervised clustering output.

## Software and dependencies

The notebook documents **Python 3.11.15 in a Conda environment** and uses:

- **Geospatial:** `rasterio`, `geopandas`, `shapely`, `GDAL` (`osgeo`), `pdal`
- **Scientific computing:** `numpy`, `pandas`, `scipy`, `opencv-python` (`cv2`)
- **Machine learning:** `scikit-learn`
- **Visualization:** `matplotlib`
- **Utilities:** `requests` and Python standard-library modules
- **Desktop GIS:** ArcGIS Pro for selected terrain and raster preparation steps

Geospatial packages such as GDAL and PDAL may require platform-specific installation. A Conda environment is recommended, but the notebook does **not** supply a tested lockfile or a complete reproducible environment specification.

## Running the notebook

1. Obtain the required imagery, LiDAR/DEM files, and authorized field-survey shapefiles.
2. Install compatible Python/geospatial dependencies in an environment that supports GDAL, PDAL, and Rasterio.
3. Open [`Version_15.ipynb`](Version_15.ipynb) in JupyterLab or Jupyter Notebook.
4. **Update all local file paths** throughout the notebook. The examples reference multiple Windows drives (`D:`, `E:`, and `G:`), and the initial directory-creation cell does not configure subsequent paths automatically.
5. Confirm each dataset's CRS, units, NoData values, spatial coverage, and predictor band order before running downstream steps.
6. Run preprocessing and stack-building cells in dependency order, followed by clustering or CART training, prediction, and evaluation.

**Resource considerations:** The notebook estimates approximately **2 TB of available disk space** for the full workflow and documents a workstation with 36 GB RAM. Processing time and memory needs depend heavily on geographic extent, resolution, and the chosen clustering method. This is a research notebook with intermediate experiments; it is **not guaranteed to execute end-to-end without adapting paths, data inputs, and individual cells**.

## Outputs

Depending on which cells are executed, the workflow produces:

- Reprojected, normalized, and clipped GeoTIFF imagery
- NDVI and HSV-derived predictor rasters
- LiDAR-derived CHMs and DTM-based slope/aspect layers
- Aligned multiband feature-stack GeoTIFFs
- K-means cluster-label rasters
- CART binary prediction rasters and optional probability rasters
- Classification evaluation reports and confusion matrices

Outputs are written to local folders specified in the notebook; they are not included in this repository by default.

## Limitations and future work

- **Spatial validation:** Evaluate model performance on geographically separated survey areas, not only a random split of sampled pixels.
- **Training data:** Refine background polygons and verify the consistency of survey dates and species labels.
- **Cluster interpretation:** Validate K-means clusters before interpreting them as species occurrences.
- **Scale sensitivity:** Compare performance by management unit, image resolution, and spatial extent.
- **Ecological constraints:** Explore masks or confidence adjustments informed by terrain, shade, soil, water, and other habitat conditions.
- **Temporal analysis:** Incorporate imagery and canopy-height data from additional years to investigate change.

These directions are discussed in the notebook's **Next Steps** section; they should not be interpreted as completed analyses.

## References

The notebook includes a literature-review and citation section covering invasive-species remote sensing, K-means clustering, and classification trees. Selected references include:

- Zhang, S., Guo, J., & Wang, Z. (2019). Combining K-means clustering and local weighted maximum discriminant projections for weed species recognition. *Frontiers in Computer Science*. https://doi.org/10.3389/fcomp.2019.00004
- Chowdhury, S., Hamerly, G., & McGarrity, M. (2024). Active learning strategy using contrastive learning and K-means for aquatic invasive species recognition. *WACVW*. https://doi.org/10.1109/WACVW60836.2024.00097
- Moisen, G. G. (2008). Classification and regression trees. *Encyclopedia of Ecology*. https://doi.org/10.1016/B978-008045405-4.00149-X

For the full bibliography and original source notes, see [`Version_15.ipynb`](Version_15.ipynb).
