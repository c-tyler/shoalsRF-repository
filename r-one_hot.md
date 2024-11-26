**About**  
This code was created in R 4.3.0.  
It is used to apply one-hot encoding to geomorphon data.  
Before processing, the raster comprised one band containing non-ordinal, categorical geomorphon type data.  
After processing, the raster comprises six bands containing binary data for the presence/absence of each geomorphon type.  
E.g., band 2 contains "1" for ridge areas and "0" for non-ridge areas.  
The raster was imported as 'geomorphon.categorical' using raster::raster()

```
library(raster)
geomorphon.onehot <- layerize(geomorphon.categorical)
names(geomorphon.onehot) <- c("flat", "ridge", "shoulder", "slope", "footslope", "valley")
```

The raster was written to the working directory in geotiff format using the command raster::writeRaster()
