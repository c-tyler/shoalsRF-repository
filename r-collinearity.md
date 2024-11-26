**About**  
This code was written in R 4.3.0.  
The purpose of the code is to test the collinearity of bathymetry and sea surface temperature rasters at the Isles of Shoals.  
Rasters are imported using the raster::raster() command (variable names indicated below parenthetically).  
The following bathymetric variables were compared:
 - Depth (depth)
 - Slope (slope)
 - Northness of Aspect (north)
 - Eastness of Aspect (east)
 - Mean Curvature (curve)
 - Terrain Ruggedness Index (tri)
 - Vector Ruggedness Measure (vrm)
(Geomorphon was not compared as it is a non-ordinal categorical variable) 
The following sea surface temperature (SST) variables were compared:
 - Median SST (med.sst)
 - Mean SST (mean.sst)
 - Standard Deviation of SST (std.sst)
 - Range of SST (range.sst)

**First Test**  
The following steps are performed in this section:
 - R packages are called
 - A pRNG seed is set for reproducibility
 - Rasters are stacked and renamed
 - NA values are omitted
 - 10,000 points are sampled
 - Collinearity among those points are tested

```
library(dplyr)
library(raster)
set.seed(1)
stack.a <- NULL
stack.a <- stack(depth, slope, north, east, curve, tri, vrm, med.sst, mean.sst, std.sst, range.sst)
names(stack.a) <- c("depth", "slope", "northness", "eastness", "curvature", "tri", "vrm", 
                    "med.sst", "mean.sst", "std.dev.sst", "range.sst")
stack.a.na <- na.omit(as.data.frame(stack.a))
sample.a <- sample_n(stack.a.na, 10000)
cor(sample.a, use = "complete.obs")
```

Collinearity was found between the following variable pairs:
 - Slope and TRI
 - TRI and VRM
 - Median SST and Mean SST
 - Standard Deviation of SST and Range of SST

**Second Test**  
TRI, Mean SST, and Range of SST were removed and collinearity was retested following the same process.

```
stack.b <- NULL
stack.b <- stack(depth, slope, north, east, curve, vrm, med.sst, std.sst)
names(stack.b) <- c("depth", "slope", "northness", "eastness", "curvature", "vrm", "med.sst", "std.dev.sst")
stack.b.na <- na.omit(as.data.frame(stack.b)
sample.b <- sample_n(stack.b.na, 10000)
cor(sample.b, use = "complete.obs")
```

No further collinearity was observed.
