# shoalsRF-repository
This repository contains code and data associated with a research paper entitled  
"Subtidal kelp habitat classification at the Isles of Shoals: an integrative Random Forest approach using bathymetry and Landsat imagery"  
A preprint is available here: http://dx.doi.org/10.2139/ssrn.5015017 

**Software, packages, and utilities**  
The following software was used in the course of this research:
- [Google Earth Engine](https://earthengine.google.com/), JavaScript web interface
- [ArcGIS Pro](https://www.esri.com/en-us/arcgis/products/arcgis-pro/overview), version 2.9.0
- [R](https://www.r-project.org/), version 4.3.0
- [Bathymetry- and Reflectivity-based Estimator of Seafloor Segments](https://www.hydroffice.org/bress/main) (BRESS), version 2.4.0

The following R packages were used in the course of this research:
- [dplyr](https://cran.r-project.org/web/packages/dplyr/index.html) (used to sort, filter, and group data)
- [ggplot2](https://cran.r-project.org/web/packages/ggplot2/index.html) (used to visualize data)
- [randomForest](https://cran.r-project.org/web/packages/randomForest/index.html) (used to produce a Random Forest model)
- [raster](https://cran.r-project.org/web/packages/raster/index.html) (used to import, export, and manage geotiff files)
- [Rmisc](https://cran.r-project.org/web/packages/Rmisc/index.html) (used to summarize datasets)
- [tidyverse](https://cran.r-project.org/web/packages/tidyverse/index.html) (used to sort, filter, and group data)

The following ArcGIS Pro utility was used in the course of this research:
- [Arc Hydro](https://www.esri.com/en-us/industries/water-resources/arc-hydro), version 2.9.111

**Datasets**  
The following datasets were used in the course of this research:
- [USGS Landsat 8 Level 2, Collection 2, Tier 1 imagery](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C02_T1_L2) (accessed through the Google Earth Engine interface)
- [WGOM Bathymetry and Backscatter: High Resolution MBES Bathymetry](https://ccom.unh.edu/project/wgom-bathbackscatter)
- [NOAA National Shoreline](https://shoreline.noaa.gov/national.html)

**Research data**  
The following ground truth file was produced based on drop camera video recordings taken at the Isles of Shoals.  
- [ground_truth.csv](https://github.com/c-tyler/shoalsRF-repository/blob/main/ground_truth.csv)
The following file is the output of our Random Forest models. It contains the training accuracy, validation accuracy, precision, recall, F1 statistic, and per-class and model-wide variable importances summarized in our publication.
- [rf_outputs.csv]
The following file contains the number of cells of each habitat class at each confidence threshold, which were used to calculate percentages in our publication.
- [classification_cells.csv]
These files are made available under a [Creative Commons Attribution 4.0 International license](https://creativecommons.org/licenses/by/4.0/).    

**Research code**  
The following Google Earth Engine JavaScript code was written and used in this research:
- [gee-landsat](https://github.com/c-tyler/shoalsRF-repository/blob/main/gee-landsat.md) (details the filtering, processing, and exporting of Landsat 8 imagery)
- [gee-standard_tools](https://github.com/c-tyler/shoalsRF-repository/blob/main/gee-standard_tools.md) (defines several functions called in gee-landsat)

The following R code was written and used in this research:
- [r-collinearity](https://github.com/c-tyler/shoalsRF-repository/blob/main/r-collinearity.md) (identifies collinearity among bathymetric and Landsat-derived sea surface temperature rasters)
- [r-one_hot](https://github.com/c-tyler/shoalsRF-repository/blob/main/r-one_hot.md) (converts a non-ordinal categorical raster to a multiband raster with binary category identifiers)
- [r-random_forest](https://github.com/c-tyler/shoalsRF-repository/blob/main/r-random_forest.md) (details the stages of training, tuning, assessing, and applying a Random Forest model)
