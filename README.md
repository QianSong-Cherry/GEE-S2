# GEE-S2
A simple tutorial to download Sentinel-2 images over a specific administrative area using Google Earth Engine platform.

## Step 0: Introduction
The tutorial is for: new users of GEE, and want to extract Sentinel-2 (or any others) time series over a certain region.
There are many useful repositories on Github.com to download remote sensing images on GEE, such as 

- [Sentinel2_GEE](https://github.com/asifatharuna/Sentinel2_GEE?tab=readme-ov-file)
- [GEE-to-NPY](https://github.com/ellaampy/GEE-to-NPY/tree/master)
- [geedim](https://github.com/leftfield-geospatial/geedim)

But as far as I know, none of them are designed for the purpose as this tutorial.

## Step 1: generate shape file of your ROI
The first step is to generate a shape file of your ROIs, which can be in various format, such as .shp. 
There are many ways to do it. One of the ways (that I used) is downloading via DIVA-GIS.

- download the shape files from [DIVA-GIS](https://diva-gis.org/) website.
- Load the shape file in [QGIS](https://qgis.org/de/site/), select the small ROI (eg. 'Bayern, Germany') and save to a shp. file.

## Step 2: setup on Google Earth Engine (GEE)
It is suggested to read some guidelines for the new users of GEE before continuing read this tutorial. Some recommandations:

- https://www.google.com/earth/outreach/learn/introduction-to-google-earth-engine/
- https://blog.csdn.net/qq_22865459/article/details/80614822

Then, upload your shape file (in step 1) to GEE Assets. And create a new project and script.

Note: to access to GEE, you need an account; to save the downloaded data, you need a Google drive account.

### Step 3: download on GEE
This tutorial is based on the one posted at https://zhuanlan.zhihu.com/p/366744507.

Step a: load the shape file
```
var roi = ee.FeatureCollection("projects/bayernforest/assets/bayern");
```
Step b: load the necessary functions
```
var batch = require('users/fitoprincipe/geetools:batch')
var GuanMethod = require("users/GHX/share:function_Library.js")
```
Step c: to mask-out clouds
```
function maskS2clouds(image){
  var qa = image.select("QA60");
  
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;
  
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
               .and(qa.bitwiseAnd(cirrusBitMask).eq(0));
  
  return image.updateMask(mask)
         
              .set('system:time_start', image.get('system:time_start'));
}
```
Step d: collect Sentinel-2 data
```
var dataset = ee.ImageCollection('COPERNICUS/S2_SR') 
                  .filterDate('2020-08-01', '2020-08-30')
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',20)) 
                  .map(maskS2clouds)
                  .filterBounds(roi); 
```
You can change the date range that you want to get a mosaic of Sentinel-2 image over the ROI. 
Set the 'CLOUDY_PIXEL_PERCENTAGE' as the rate of cloud coverage.
Step e: get a mosaic
```
var S2Dataset = GuanMethod.ic_Mosaic_sameDate(dataset)
                          .map(function(image){
                            return image.clip(roi)
                          })
```
Step f: save the data to Google drive
```
batch.Download.ImageCollection.toDrive(S2Dataset,"result", {
scale: 10,
region: roi,          
maxPixels:34e10,          
type:"int16" });
```