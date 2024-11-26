**About**  
This code was created in Google Earth Engine using JavaScript.  
It details the steps of collecting and sorting Landsat 8 imagery, processing it, and exporting sea surface temperature data at the Isles of Shoals.  
Several functions are mapped in this script. They are described in [gee-standard_tools](https://github.com/c-tyler/shoalsRF-repository/blob/main/gee-standard_tools.md).  
They were stored in a separate script and called using the require() function.

**Import and filtering**  
The following steps are performed in this section:
  - The dataset USGS Landsat 8 Level 2, Collection 2, Tier 1 is imported
  - The dataset is filtered to area of interest, months and years of interest, and Landsat scene (Path 12, Row 30) of interest

```
var ls8 = ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")
var roi = shoals;
var collated = ls8.filterBounds(roi)
                  .filter(ee.Filter.calendarRange(7,9,'month'))
                  .filter(ee.Filter.calendarRange(2019,2023, 'year'))
                  .filter(ee.Filter.eq('WRS_PATH', 12))
                  .filter(ee.Filter.eq('WRS_ROW', 30))
```

**Imagery processing**  
The following steps are performed in this section:
 - The thermal infared band is converted to degrees Celsius
 - The images are clipped to the region of interest
 - Clouds are masked from each image
 - The percentage of cloud cover in each clipped image is recorded
 - Images with greater than 25% cloud cover are removed from the collection

```
var processed = collated.map(standard_tools.kelfixer)
                  .map(function(image){
                        return image.clip(roi)
                        })
                  .map(standard_tools.cloudmasker)
                  .map(function(image){
                        return standard_tools.localscore(image, roi)
                        })
                  .filter(ee.Filter.lte('LOCAL_CLOUD_PERCENT', 25));
```

**Image reduction and export**  
In this section, the following steps are performed:
 - The image collection is reduced to a single multiband image containing several summary statistics for each cell
 - This image is exported to Google Drive

```
var crunched = standard_tools.TIRScruncher(processed)
                             .clip(roi);
Export.image.toDrive({
              image: crunched,
              description: 'islesOfShoalsTemperatures',
              region: roi,
              folder: 'output',
              scale: 30
});
```
