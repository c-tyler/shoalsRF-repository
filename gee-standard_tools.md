**About**  
These functions were created in Google Earth Engine using JavaScript.  
They are used in [gee-landsat](https://github.com/c-tyler/shoalsRF-repository/blob/main/gee-landsat.md).  
They were stored in a separate script and called using the require() function.  
Each function is intended to work with images or image collections drawing from this dataset: [USGS Landsat 8 Level 2, Collection 2, Tier 1](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C02_T1_L2).  
Within Google Earth Engine, the data can be imported from the following location:

```
ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")
```

**kelfixer**  
This function is intended to be mapped over a collection of images.  
The function performs the following steps:
 - It identifies the multiplicative and additive scaling factors associated with the thermal infrared band (redundant, as these are constant across the dataset)
 - It applies these factors to scale the INT16 representation of Kelvins to decimal values
 - It converts Kelvins to degrees Celsius, rounding to 4 decimal places
 - It returns the original image with the degrees Celsius band appended

```
exports.kelfixer = function(image){
      var multfactor = image.getNumber('TEMPERATURE_MULT_BAND_ST_B10');
      var offset = image.getNumber('TEMPERATURE_ADD_BAND_ST_B10');
      var celsconv = image.select('ST_B10')
                      .multiply(multfactor)
                      .add(offset)
                      .subtract(273.15)
                      .multiply(10000)
                      .round()
                      .divide(10000)
                      .rename('degCel');
      return image.addBands(celsconv);
};
```

**cloudmasker**  
This function is intended to be mapped over a collection of images.
The function performs the following steps:
 - It calls the quality assurance band 'QA_PIXEL'
 - It selects bit 3, which identifies cloud cover
 - It creates a mask to remove cells where clouds are identified
 - It returns the original image with the mask applied

```
exports.cloudmasker = function(image){
      var qaband = image.select('QA_PIXEL');
      var cloudbit = 1 << 3
      var skymask = qaband.bitwiseAnd(cloudbit)
                          .eq(0);
      return image.updateMask(skymask);
};
```

**localscore**  
This function is intended to be mapped over a collection of images after **cloudmasker** has been used.  
The function performs the following steps:
 - It identifies the scale and coordinate reference system of the image
 - It counts the amount of cells currently present (after the cloud mask was applied)
 - It removes the cloud mask and counts the total number of cells
 - It calculates the percentage of cells that were masked
 - It returns the original image with this percentage included as a property called 'LOCAL_CLOUD_PERCENT'

```
exports.localscore = function(image, roi){
    var nomscale = image.projection()
                        .nominalScale();
    var coord = image.projection()
                     .crs();
    var cloudcount = image.select('degCel')
                          .reduceRegion({
                              reducer: ee.Reducer.count(),
                              geometry: roi,     
                              scale: nomscale,
                              crs: coord,
                              maxPixels: 1e9
                              })
                          .get('degCel');
    var totalcount = image.select('degCel')
                          .unmask()
                          .reduceRegion({
                              reducer: ee.Reducer.count(),
                              geometry: roi,
                              scale: nomscale,
                              crs: coord,
                              maxPixels: 1e9
                              })
                          .get('degCel');
    var cloudpercent = ee.Number(1).subtract(ee.Number(cloudcount).divide(totalcount))
                                   .multiply(100);
    return image.set('LOCAL_CLOUD_PERCENT', cloudpercent)
};
```

**TIRScruncher**  
This function is intended to be applied to a collection of images to reduce it to a single raster of summary statistics.  
The function performs the following steps:
 - It records the first and last acquisition dates in the collection
 - It records the amount of images in the collection
 - It calculates the mean 'LOCAL_CLOUD_PERCENT' of the collection (from function **localscore**)
 - It creates a reducer that will calculate the median, mean, standard deviation, raster count, minimum, and maximum value of each band
 - It selects only the 'degCel' band (from function **kelfixer**) and applies the reducer to that band
 - It separates and renames the reduced bands
 - It subtracts the minimum value from the maximum value to identify a temperature range
 - It converts the raster count band (named 'valid_cells') to double format (multiband images must have uniform data types to be exported)
 - It produces an image with the median, mean, standard deviation, range, and raster count bands
 - It returns this image with the first date, last date, collection size, and mean cloud percent properties attached

```
exports.TIRScruncher = function(imageCollection){
      var firstday = imageCollection.sort('DATE_ACQUIRED')
                                    .first()
                                    .get('DATE_ACQUIRED');
      var lastday = imageCollection.sort('DATE_ACQUIRED', false)
                                    .first()
                                    .get('DATE_ACQUIRED');
      var collength = imageCollection.size();
      var meancloud = imageCollection.aggregate_mean('LOCAL_CLOUD_PERCENT')
                                     .multiply(1000)
                                     .round()
                                     .divide(1000);
      var multiReducer = ee.Reducer.median()
                                    .combine({
                                          reducer2: ee.Reducer.mean(),
                                          sharedInputs: true
                                          })
                                    .combine({
                                          reducer2: ee.Reducer.stdDev(),
                                          sharedInputs: true
                                          })
                                    .combine({
                                          reducer2: ee.Reducer.count(),
                                          sharedInputs: true
                                          })
                                    .combine({
                                          reducer2: ee.Reducer.minMax(),
                                          sharedInputs: true
                                          });
      var collapsed = imageCollection.select('degCel')
                                     .reduce({reducer: multiReducer})
      var bandcut = collapsed.select('degCel_median', 'degCel_mean', 'degCel_stdDev', 'degCel_max', 'degCel_min')
                             .rename('ST_median', 'ST_mean', 'ST_stdDev', 'ST_max', 'ST_min');
      var bandmath = collapsed.select('degCel_max')
                              .subtract(collapsed.select('degCel_min'))
                              .rename('ST_range');
      var count_int = collapsed.select('degCel_count')
                               .rename('valid_cells')
                               .toDouble();
      var combined = bandcut.addBands(bandmath)
                            .addBands(count_int);
      var image = combined.set('ACQUISITION_LAST', lastday)
                          .set('ACQUISITION_FIRST', firstday)
                          .set('IMAGE_TOTAL', collength)
                          .set('LOCAL_CLOUD_PERCENT_MEAN', meancloud)
      return image;
};
```
