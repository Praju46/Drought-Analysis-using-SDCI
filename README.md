# Drought-Analysis-using-SDCI
/****
 * Drought Monitoring using SDCI (Standardized Drought Condition Index)
 * Data Sources: MODIS (NDVI, LST), CHIRPS (Precipitation)
 * Developed by: Mali Prajakta Bhimrao
 ****/

// Define Region of Interest (ROI)
var roi = ee.Geometry.Rectangle([xmin, ymin, xmax, ymax]);

// Load MODIS NDVI and LST Data
var ndvi = ee.ImageCollection("MODIS/006/MOD13A2")
    .filterBounds(roi)
    .filterDate("2023-01-01", "2023-12-31")
    .select("NDVI")
    .map(function(img) { return img.multiply(0.0001).copyProperties(img, ["system:time_start"]); });

var lst = ee.ImageCollection("MODIS/006/MOD11A2")
    .filterBounds(roi)
    .filterDate("2023-01-01", "2023-12-31")
    .select("LST_Day_1km")
    .map(function(img) { return img.multiply(0.02).subtract(273.15).copyProperties(img, ["system:time_start"]); });

// Load CHIRPS Precipitation Data
var precip = ee.ImageCollection("UCSB-CHG/CHIRPS/DAILY")
    .filterBounds(roi)
    .filterDate("2023-01-01", "2023-12-31")
    .select("precipitation");

// Compute VCI (Vegetation Condition Index)
var ndvi_min = ndvi.reduce(ee.Reducer.min());
var ndvi_max = ndvi.reduce(ee.Reducer.max());
var vci = ndvi.map(function(img) {
    return img.subtract(ndvi_min).divide(ndvi_max.subtract(ndvi_min)).rename("VCI");
});

// Compute TCI (Temperature Condition Index)
var lst_min = lst.reduce(ee.Reducer.min());
var lst_max = lst.reduce(ee.Reducer.max());
var tci = lst.map(function(img) {
    return lst_max.subtract(img).divide(lst_max.subtract(lst_min)).rename("TCI");
});

// Compute PCI (Precipitation Condition Index)
var precip_min = precip.reduce(ee.Reducer.min());
var precip_max = precip.reduce(ee.Reducer.max());
var pci = precip.map(function(img) {
    return img.subtract(precip_min).divide(precip_max.subtract(precip_min)).rename("PCI");
});

// Combine Indices to Compute SDCI
var sdci = pci.combine(vci).combine(tci).map(function(img) {
    return img.expression(
        "0.5 * PCI + 0.25 * VCI + 0.25 * TCI",
        {
            PCI: img.select("PCI"),
            VCI: img.select("VCI"),
            TCI: img.select("TCI")
        }
    ).rename("SDCI");
});

// Visualization Parameters
var vizParams = {
    min: 0,
    max: 1,
    palette: ["red", "yellow", "green"]
};

// Display on Map
Map.centerObject(roi, 6);
Map.addLayer(sdci.median(), vizParams, "SDCI (2023)");
