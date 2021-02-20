---
title: "CEDEN & SURF Merge"
author: "Skyler Elmstrom"
date: "2/17/2021"
output:
  html_document:
    code_download: true
    keep_md: true
---




```r
library(data.table)
library(lubridate)
library(sf)
library(tidyverse)
```

<br>

### SURF Data {.tabset}

SURF Data was acquired at the [DPR SURF database web](https://www.cdpr.ca.gov/docs/emon/surfwtr/surfcont.htm) page as CSVs via FTP download on 2/17/2021:

* ftp://transfer.cdpr.ca.gov/pub/outgoing/EM/Surface/SURF_water.csv (309.8 MB)
* ftp://transfer.cdpr.ca.gov/pub/outgoing/EM/Surface/SURF_SED.csv (52.2 MB)

According to email correspondence with Dr. Xuyang Zhang at CDPR on 2/17/2021, latitude and longitude data are in WGS84.

<br>

#### Load Data


```r
### Read SURF Tables
SURF.Sed <- fread("Data/SURF_SED.csv")
SURF.Sed.sf <- st_as_sf(SURF.Sed, coords = c("Longitude", "Latitude"), remove = F, crs = "WGS84")

SURF.WQ <- fread("Data/SURF_water.csv")
SURF.WQ.sf <- st_as_sf(SURF.WQ, coords = c( "Longitude", "Latitude"), remove = F, crs = "WGS84")

### Load Risk Regions from GitHub CEDEN repository (change if moved)

USFE.RiskRegions.z <- "https://github.com/WWU-IETC-R-Collab/CEDEN-mod/raw/main/Data/USFE_RiskRegions_9292020.zip"

unzip_shape <- function(InputShapeZip){
  dl.temp <- tempfile() # Create local temp file for zipped shapefile
  dl.temp2 <- tempfile() # Create a second local temp file to store unzipped shapefile
  download.file(InputShapeZip, dl.temp, quiet=T) # Downloads zip file from InputShape
  unzip(zip = dl.temp, exdir = dl.temp2) # Unzips zip file
  shapefile.out <-list.files(dl.temp2, pattern = ".shp$",full.names=TRUE) # stores file path of files with .shp ext in dl.temp2
  sf::st_read(shapefile.out) # Reads shapefile as sf object
}

USFE.RiskRegions <- unzip_shape(USFE.RiskRegions.z) # CRS is WGS 84
```
 
<br>

#### Spatial/Temporal Query


```r
# SURF Sed
SURF.Sed.sf <- st_join(SURF.Sed.sf, USFE.RiskRegions[1], left = T) %>%
  filter(!is.na(Subregion)) %>%
  filter(between(Sample_date, as_date("2009-10-01"), as_date("2019-09-30")))

#SURF Water
SURF.WQ.sf <- st_join(SURF.WQ.sf, USFE.RiskRegions[1], left = T) %>%
  filter(!is.na(Subregion)) %>%
  filter(between(Sample_date, as_date("2009-10-01"), as_date("2019-09-30")))

# Export SURF Tables
write_csv(SURF.WQ.sf, "Data/Output/SURFMod_water.csv") # Note: coerces empty data fields to NA
write_csv(SURF.Sed.sf, "Data/Output/SURFMod_SED.csv") # Note: coerces empty data fields to NA
```

<br><br>

### CEDEN & SURF

<br>

#### Check CEDEN & SURF for Duplicate Data


```r
# Load CEDEN Data
CEDENMod_Tox <- fread("https://github.com/WWU-IETC-R-Collab/CEDEN-mod/raw/main/Data/Output/CEDENMod_Toxicity.csv")
CEDENMod_WQ <- fread("https://github.com/WWU-IETC-R-Collab/CEDEN-mod/raw/main/Data/Output/CEDENMod_WQ.csv")
```
