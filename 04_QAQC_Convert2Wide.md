---
title: "Convert to Wide"
author: "Erika W"
date: "4/5/2021"
output:
  html_document:
    code_download: true
    keep_md: true
    toc: true
    toc_float:
      toc_collapsed: true
    toc_depth: 3
    theme: lumen
---





```r
rm(list = ls())
library(data.table)
library(lubridate)
library(sf)
library(tidyverse)
```


## Introduction

Because NETICA requires the nodes to be columns, this data needs to be transformed to wide format.

1. Start with the integrated, modified CEDEN and SURF datasets that have had the conceptual model categories appended
2. Convert units to be consistent within each analyte category (node)
3. Summarize into wide format

## 1. Load Data

```r
# Load data
CEDENSURF <- fread("https://github.com/WWU-IETC-R-Collab/CEDENSURF-mod/raw/main/Data/Output/CEDENSURF_Limited.csv") %>% select(Analyte, Result, Unit,CollectionMethod, Matrix, Date, Subregion, StationName, Latitude, Longitude,SelectList)
```

<br/>

## 2. Convert Units & Names

Units should be consistent within each analyte-Matrix combination. Initially, they were not. I separated the dataframe into categories of the conceptual model, and worked within those WideSubsets to unify units of each analyte-matrix combination.

Because Netica cannot allow spaces, -, /, or other symbols in the column names, analyte names must also be converted to usable formats.


```r
CEDENSURF %>% group_by(Analyte, Matrix) %>%
  summarize(n_unit = n_distinct(Unit)) %>% head(.)
```

```
## `summarise()` has grouped output by 'Analyte'. You can override using the `.groups` argument.
```

```
## # A tibble: 6 x 3
## # Groups:   Analyte [4]
##   Analyte            Matrix   n_unit
##   <chr>              <chr>     <int>
## 1 atrazine           sediment      1
## 2 atrazine           water         3
## 3 atrazine degradate water         1
## 4 bifenthrin         sediment      3
## 5 bifenthrin         water         4
## 6 chlorpyrifos       sediment      2
```

### WQP  {.tabset}

Few discrepancies in units between analytes in each category:

1. Phosphorous; 33/900 records had mg/Kg dw rather than mg/L

2. Turbidity; One analyte is specifically turbidity, units are NTU. Some variables that may also indicate turbidity had different units, but are possibly not comparable.

No conversions were made for WQP Parameters, though some analytes were omitted.


```r
WQP <- CEDENSURF %>% filter(SelectList == "WQP") 
```

#### Univariate - names and units as expected

Temp - *C
pH - no units

#### Oxygen - mg/L

**water** (3504 records) mg/L
**sediment** (281 records) mg/L

89 water records were converted from %DO, assuming an average water temperature of 20C.
    *Oxygen mg/L = (% Saturation)/10.995*
    https://www.waterontheweb.org/under/waterquality/dosatcalc.html


```r
# Oxygen - 
# Water Units = 3,416 records in mg/L and 89 in %
# Sediment Unit = 281 records mg/L

# Convert % to mg/L

WQP$Result[WQP$Analyte == "oxygen" & WQP$Unit == "%"] <- WQP$Result[WQP$Analyte == "oxygen" & WQP$Unit == "%"] /10.995

# Correct units

WQP$Unit[WQP$Analyte == "oxygen"] <- "mg/L"

# Check

WQP %>% filter(Analyte == "oxygen") %>% filter(Matrix == "water") %>% distinct(Unit)

WQP %>% filter(Analyte == "oxygen") %>% filter(Matrix == "water") %>% nrow(.)
```

#### Nitrogen

*Water* - mg/L
nitrate as n (379 records)
nitrite as n (444 records)

Total N:
nitrogen (709 records)
nitrate + nitrite as n (844 records)

*Sediment* - % dw
nitrogen (33 records)


```r
# Nitrate - Unit = mg/L (nitrate as n)

WQP %>% filter(Analyte== "nitrate as n") %>% distinct(Unit)
WQP$Analyte[WQP$Analyte == "nitrate as n"] <- "nitrate" #rename to omit spaces

# Nitrite  - Unit = mg/L (nitrite as n) 

WQP %>% filter(Analyte== "nitrite as n") %>% distinct(Unit)
WQP$Analyte[WQP$Analyte == "nitrite as n"] <- "nitrite"

# Combined as Nitrogen - 748 records; mg/L and %dw

    # Nitrogen/sediment is in % dw

    WQP %>% filter(grepl("nitro", Analyte)) %>% 
            filter(Matrix == 'sediment') %>% distinct(Unit)

    # Nitrogen/water is in mg/L
    
    WQP %>% filter(grepl("nitro", Analyte)) %>% 
            filter(Matrix == 'water') %>% distinct(Unit)
    
# Combined as Nitra + Nitrite - 844 records; mg/L (all water)

    WQP %>% filter(Analyte == "nitrate + nitrite as n") %>% 
            filter(Matrix == 'water') %>% distinct(Unit)
    # rename
    WQP$Analyte[WQP$Analyte == "nitrate + nitrite as n"] <- "nitrogen" 
```

#### Phosphorous 

**water** (819 records) mg/L
**sediment** (27 records) mg/Kg dw


```r
# Phosphorous ("phosphorous as p")

# Water - 819 records (mg/L)

WQP %>% filter(grepl("phos", Analyte)) %>% 
            filter(Matrix == 'water') %>% distinct(Unit)

# Sediment - 27 records (mg/Kg dw)

WQP %>% filter(grepl("phos", Analyte)) %>% 
            filter(Matrix == 'sediment') %>% distinct(Unit)

# Correct Name

WQP$Analyte[WQP$Analyte == "phosphorus as p"] <- "phosphorus"
```

#### Turbidity - NTU

Units differ between three analytes that might refer to turbidity:
    # 1,600 turbidity (NTU), 11 settleable solids (mL/L/hr), 
    # 94 suspended sediment concentration (mg/L)
    
I chose to remove all but Turbidity from our dataset.


```r
# Turbidity

    ## Units differ between three analytes that might refer to turbidity:
    # 1,600 turbidity (NTU), 11 settleable solids (mL/L/hr), 
    # 94 suspended sediment concentration (mg/L)
    
    WQP %>% filter(grepl("turb", Analyte)) %>% distinct(Unit)
    
    ## Chose to remove the other two from our DF
    WQP<- WQP %>% filter(!grepl("settle", Analyte)) %>%
                  filter(!grepl("suspend", Analyte))
```

#### Salinity - ppt

Converted all to ppt


```r
# Salinity
  # Check units:
  # WQP %>% filter(grepl("salinity", Analyte))%>%distinct(Unit)

    # ppt and psu are equivalent measures. Convert all to ppt
    
    # Run conversion on subset of data
    Sal <- WQP %>% filter(Analyte == "salinity") %>% 
            mutate(Unit = "ppt") 
    
    # Remove that subset from Main df
    WQP<- WQP %>% filter(!Analyte == "salinity")
    
    # Bind converted back to Main df
    WQP <- rbind(WQP, Sal)
```

#### ElectricalConductivity

Units are in umhos/cm and in uS/cm

A mho per meter (mho/m) is an older unit of electrical conductivity (also known as specific conductance). The mho is the reciprocal of the ohm. Though siemens was introduced in the late 1970s, this unit can still be found in some old measurement instruments. 1 mho/m = 1 S/m.

1 mho/m = 1 S/m

Therefore umhos/cm and uS/cm are equivalent units

```r
# Correct name for all units (equivalent)
WQP$Unit[WQP$Analyte == "electricalconductivity"] <- "uS/cm"
```

#### **WQP Result**


```r
WQP %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```

```
## # A tibble: 19 x 5
## # Groups:   Analyte, Matrix [19]
##    Analyte                Matrix   Unit         n      mean
##    <chr>                  <chr>    <chr>    <int>     <dbl>
##  1 electricalconductivity sediment uS/cm        3  469.    
##  2 electricalconductivity water    uS/cm       94   NA     
##  3 nitrate                water    mg/L       379    0.619 
##  4 nitrite                water    mg/L       444    0.0234
##  5 nitrogen               sediment % dw        39    0.102 
##  6 nitrogen               water    mg/L      1553    1.44  
##  7 oxygen                 sediment mg/L       281    7.43  
##  8 oxygen                 water    mg/L      3504   NA     
##  9 ph                     sediment none       281    7.99  
## 10 ph                     water    none      3359   NA     
## 11 phosphorus             sediment mg/Kg dw    27  535.    
## 12 phosphorus             water    mg/L       819    0.321 
## 13 salinity               sediment ppt         17   20.7   
## 14 salinity               water    ppt        353    3.58  
## 15 sodium                 sediment mg/Kg dw     7 1676.    
## 16 sodium                 water    mg/L        17   19.9   
## 17 temperature            sediment Deg C      244   22.2   
## 18 temperature            water    Deg C     3456   NA     
## 19 turbidity              water    NTU       1639   NA
```


```r
Wide.WQP.Waterdf <- WQP %>% filter(!Matrix == "sediment") %>%
  group_by(Date, Latitude, Longitude, Analyte, Matrix) %>%
  summarize(Subregion = first(Subregion),
            Mean = mean(Result, na.rm = T)) %>%
  pivot_wider(names_from = Analyte,
              names_repair = "check_unique",
              values_from = Mean) # Values to fill columns
```

```
## `summarise()` has grouped output by 'Date', 'Latitude', 'Longitude', 'Analyte'. You can override using the `.groups` argument.
```



```r
write.csv(x = Wide.WQP.Waterdf , file = "Data/Output/WideSubsets/WQP.Wide.water.csv", 
          row.names = F)
```

<br>

### Metals  {.tabset}

Metals were measured in ug/L and mg/Kg dw. Mercury also had few measures in ng/L and ug/Kg - these values were divided by 1000 to convert to ug/L and mg/Kg. 

**water**
mercury ug/L 
selenium (208 records) mg/L

**sediment**

mercury (mg/Kg dw)
selenium (38 records) mg/Kg dw


```r
Metal <- CEDENSURF %>% filter(SelectList == "Metal")

# Selenium

    # Water - 208 records, mg/L
    Metal %>% filter(grepl("selenium", Analyte))  %>% 
              filter(Matrix == 'water') %>% distinct(Unit)

    # Sediment - 38 records, mg/Kg dw
    Metal %>% filter(grepl("selenium", Analyte))  %>% 
               filter(Matrix == 'sediment') %>% distinct(Unit)

# Mercury

    # Water - ug/L & ng/L
    
    Metal %>% filter(grepl("mercury", Analyte))%>% 
              filter(Matrix == 'water') %>% distinct(Unit)
    
        # Convert ng to ug
          Metal$Result[
            Metal$Analyte == "mercury" & Metal$Unit == "ng/L"] <- Metal$Result[
              Metal$Analyte == "mercury" & Metal$Unit == "ng/L"]/1000
          
        # Correct units
          Metal$Unit[Metal$Analyte == "mercury" & Metal$Matrix == "water"] <- "ug/L"
    
          
    # Sediment - (19) ug/Kg dw	& (57) mg/Kg dw
    
    Metal %>% filter(grepl("mercury", Analyte))%>% 
              filter(Matrix == 'sediment') %>% distinct(Unit)

            # Convert ug to mg
              Metal$Result[
                Metal$Analyte == "mercury" & Metal$Unit == "ug/Kg dw"] <- Metal$Result[
                  Metal$Analyte == "mercury" & Metal$Unit == "ug/Kg dw"]/1000
              
            # Correct units
              Metal$Unit[Metal$Analyte == "mercury" &
                           Metal$Matrix == "sediment"] <- "mg/Kg dw"
```


#### **Metal Result**


```r
Metal %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```

```
## # A tibble: 4 x 5
## # Groups:   Analyte, Matrix [4]
##   Analyte  Matrix   Unit         n    mean
##   <chr>    <chr>    <chr>    <int>   <dbl>
## 1 mercury  sediment mg/Kg dw    76 0.111  
## 2 mercury  water    ug/L       215 0.00545
## 3 selenium sediment mg/Kg dw    38 0.251  
## 4 selenium water    ug/L       208 0.302
```



```r
Wide.Metal.Waterdf <- Metal %>% filter(!Matrix == "sediment") %>%
  group_by(Date, Latitude, Longitude, Analyte, Matrix) %>%
  summarize(Subregion = first(Subregion),
            Mean = mean(Result, na.rm = T)) %>%
  pivot_wider(names_from = Analyte,
              names_repair = "check_unique",
              values_from = Mean) # Values to fill columns
```

```
## `summarise()` has grouped output by 'Date', 'Latitude', 'Longitude', 'Analyte'. You can override using the `.groups` argument.
```

```r
write.csv(x = Wide.Metal.Waterdf , file = "Data/Output/WideSubsets/Metal.Wide.water.csv", 
          row.names = F)
```

<br>

### Organophosphates  {.tabset}


```r
OrganoP <- CEDENSURF %>% filter(SelectList == "OrganoP")
```

#### chlorpyrifos - ppb


```r
# chlorpyrifos

  # Sediment:	ppb	331	, ng/g dw	71	(equivalent units)
    OrganoP$Unit[OrganoP$Analyte == "chlorpyrifos" &
                           OrganoP$Matrix == "sediment"] <- "ppb"

  # Water:	ng/L	220, pg/L	5, ppb	1275	
      # ppb = pg / (1000*1000)
      # ppb = ng/L / 1000
    
      # Convert ng/L to ppb
        OrganoP$Result[OrganoP$Analyte == "chlorpyrifos" &
                  OrganoP$Unit == "ng/L"] <- OrganoP$Result[
                    OrganoP$Analyte == "chlorpyrifos" &
                    OrganoP$Unit == "ng/L"] /1000
      # Convert pg/L to ppb
            OrganoP$Result[OrganoP$Analyte == "chlorpyrifos" &
                  OrganoP$Unit == "pg/L"] <- OrganoP$Result[
                    OrganoP$Analyte == "chlorpyrifos" &
                    OrganoP$Unit == "pg/L"] /(1000*1000)
      # Correct units
       OrganoP$Unit[OrganoP$Analyte == "chlorpyrifos" &
                    OrganoP$Matrix == "water"] <- "ppb"
       
OrganoP %>% filter(Analyte== "chlorpyrifos") %>% distinct(Unit)
```

#### diazinon - ppb

diazinon (2239 records) ppb

diazinon degradate (95 records) ppb

diazinon oxon (14 records) ppb

diazoxon (234 records) ppb

When diazinon enters the body, it is oxidatively decomposed to diazoxon, an organophosphate compound that is much more poisonous than diazinon; it mainly causes the inhibition of AChE.

All three analytes (diazinon oxon, diazoxon, and diazinon degradate) refer to this oxidized degradate, and were therefore renamed for consistency to "diazoxon"


```r
# diazinon

  # Sediment:	ppb	and ng/g dw	(equivalent units)
    OrganoP$Unit[OrganoP$Analyte == "diazinon" &
                           OrganoP$Matrix == "sediment"] <- "ppb"

  # Water:	pg/L, ng/L, ug/L, ppb
    
      # ppb = pg / (1000*1000)
      # ppb = ng/L / 1000
      # ppb = ug/L
    
      # Convert ng/L to ppb
        OrganoP$Result[OrganoP$Analyte == "diazinon" &
                  OrganoP$Unit == "ng/L"] <- OrganoP$Result[
                    OrganoP$Analyte == "diazinon" &
                    OrganoP$Unit == "ng/L"] /1000
      # Convert pg/L to ppb
        OrganoP$Result[OrganoP$Analyte == "diazinon" &
              OrganoP$Unit == "pg/L"] <- OrganoP$Result[
                OrganoP$Analyte == "diazinon" &
                OrganoP$Unit == "pg/L"] /(1000*1000)
      # Correct units
         OrganoP$Unit[OrganoP$Analyte == "diazinon" &
                      OrganoP$Matrix == "water"] <- "ppb"
```


```r
# Diazoxon

    # Assess coverage
    OrganoP %>% filter(Analyte == "diazoxon") %>%
      distinct(Subregion) # coverage in 3 regions
    
    # Convert ng/L to ppb
      OrganoP$Result[OrganoP$Analyte == "diazoxon" &
                OrganoP$Unit == "ng/L"] <- OrganoP$Result[
                  OrganoP$Analyte == "diazoxon" &
                  OrganoP$Unit == "ng/L"] /1000
    # Correct units
      OrganoP$Unit[OrganoP$Analyte == "diazoxon"] <- "ppb"

# Diazinon degradate
      
    # Assess coverage
    OrganoP %>% filter(Analyte == "diazinon degradate") %>%
      distinct(Subregion) # all 95 records in Sacramento river.
    # Units
    OrganoP %>% filter(Analyte == "diazinon degradate") %>%
      distinct(Unit) # all 95 records in Sacramento river.
    # Rename
    OrganoP$Analyte[OrganoP$Analyte == "diazinon degradate"] <-
      "diazoxon" 

# Diazinon oxon           
    
      # Convert ng/L to ppb 
      OrganoP$Result[OrganoP$Analyte == "diazinon oxon" &
                OrganoP$Unit == "ng/L"] <- OrganoP$Result[
                  OrganoP$Analyte == "diazinon oxon" &
                  OrganoP$Unit == "ng/L"] /1000
      
    # Correct units
      OrganoP$Unit[OrganoP$Analyte == "diazinon oxon"] <- "ppb"
      
    # Rename
      OrganoP$Analyte[OrganoP$Analyte == "diazinon oxon"] <-
        "diazoxon"
```

#### malathion - ppb


```r
# malathion

  # Sediment:	ppb	and ng/g dw	(equivalent units)
    OrganoP$Unit[OrganoP$Analyte == "malathion" &
                           OrganoP$Matrix == "sediment"] <- "ppb"

  # Water: ng/L, ug/L, ppb
    
      # ppb = ng/L / 1000
      # ppb = ug/L
    
      # Convert ng/L to ppb
        OrganoP$Result[OrganoP$Analyte == "malathion" &
                  OrganoP$Unit == "ng/L"] <- OrganoP$Result[
                    OrganoP$Analyte == "malathion" &
                    OrganoP$Unit == "ng/L"] /1000
      
      # Correct units
         OrganoP$Unit[OrganoP$Analyte == "malathion" &
                      OrganoP$Matrix == "water"] <- "ppb"

OrganoP %>% filter(Analyte== "malathion") %>% distinct(Unit)
```

```
##    Unit
## 1:  ppb
```


#### **OrganoP Result**

```r
OrganoP %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```

```
## # A tibble: 8 x 5
## # Groups:   Analyte, Matrix [8]
##   Analyte      Matrix   Unit      n     mean
##   <chr>        <chr>    <chr> <int>    <dbl>
## 1 chlorpyrifos sediment ppb     402  0.101  
## 2 chlorpyrifos water    ppb    2229  0.00731
## 3 diazinon     sediment ppb     361 -0.0160 
## 4 diazinon     water    ppb    1878  0.00677
## 5 diazoxon     sediment ppb      48  0      
## 6 diazoxon     water    ppb     295  0      
## 7 malathion    sediment ppb     328 -0.0677 
## 8 malathion    water    ppb    1617  0.00579
```


```r
Wide.OrganoP.Waterdf <- OrganoP %>% filter(!Matrix == "sediment") %>%
  group_by(Date, Latitude, Longitude, Analyte, Matrix) %>%
  summarize(Subregion = first(Subregion),
            Mean = mean(Result, na.rm = T)) %>%
  pivot_wider(names_from = Analyte,
              names_repair = "check_unique",
              values_from = Mean) # Values to fill columns
```

```
## `summarise()` has grouped output by 'Date', 'Latitude', 'Longitude', 'Analyte'. You can override using the `.groups` argument.
```

```r
write.csv(x = Wide.OrganoP.Waterdf , file = "Data/Output/WideSubsets/OrganoP.Wide.water.csv", 
          row.names = F)
```

<br>

### Pyrethroids  {.tabset}


```r
Pyre <- CEDENSURF %>% filter(SelectList == "Pyrethroids")

Pyre %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```

```
## # A tibble: 26 x 5
## # Groups:   Analyte, Matrix [10]
##    Analyte    Matrix   Unit         n         mean
##    <chr>      <chr>    <chr>    <int>        <dbl>
##  1 bifenthrin sediment ng/g dw    111    17.0     
##  2 bifenthrin sediment ppb        390     3.87    
##  3 bifenthrin sediment ug/Kg dw    21     0.118   
##  4 bifenthrin water    ng/L       275     0.662   
##  5 bifenthrin water    pg/L         8 77162.      
##  6 bifenthrin water    ppb        662     0.00133 
##  7 bifenthrin water    ug/L        73     0.000347
##  8 cyfluthrin sediment ng/g dw    111     2.05    
##  9 cyfluthrin sediment ppb        390     0.486   
## 10 cyfluthrin sediment ug/Kg dw    21     0       
## # ... with 16 more rows
```

#### cyfluthrin - ppb


```r
# cyfluthrin

  # Sediment:	ppb	and ng/g dw	(equivalent units)
    Pyre$Unit[Pyre$Analyte == "cyfluthrin" &
              Pyre$Matrix == "sediment"] <- "ppb"

  # Water: pg/L, ng/L, ug/L, ppb
      # ppb = pg/L /(1000*1000)
      # ppb = ng/L / 1000
      # ppb = ug/L
      
      # Convert pg/L to ppb
        Pyre$Result[Pyre$Analyte == "cyfluthrin" &
              Pyre$Unit == "pg/L"] <- Pyre$Result[
                Pyre$Analyte == "cyfluthrin" &
                Pyre$Unit == "pg/L"] /(1000*1000)
        
      # Convert ng/L to ppb
        Pyre$Result[Pyre$Analyte == "cyfluthrin" &
                  Pyre$Unit == "ng/L"] <- Pyre$Result[
                    Pyre$Analyte == "cyfluthrin" &
                    Pyre$Unit == "ng/L"] /1000
      
      # Correct units
         Pyre$Unit[Pyre$Analyte == "cyfluthrin" &
                      Pyre$Matrix == "water"] <- "ppb"

Pyre %>% filter(Analyte== "cyfluthrin") %>% distinct(Unit)
```
#### bifenthrin


```r
# bifenthrin

  # Sediment:	ug/Kg dw, ppb, ng/g dw (equivalent units)
    Pyre$Unit[Pyre$Analyte == "bifenthrin" &
              Pyre$Matrix == "sediment"] <- "ppb"

  # Water: pg/L, ng/L, ug/L, ppb
      # ppb = pg/L /(1000*1000)
      # ppb = ng/L / 1000
      # ppb = ug/L
      
      # Convert pg/L to ppb
        Pyre$Result[Pyre$Analyte == "bifenthrin" &
              Pyre$Unit == "pg/L"] <- Pyre$Result[
                Pyre$Analyte == "bifenthrin" &
                Pyre$Unit == "pg/L"] /(1000*1000)
        
      # Convert ng/L to ppb
        Pyre$Result[Pyre$Analyte == "bifenthrin" &
                  Pyre$Unit == "ng/L"] <- Pyre$Result[
                    Pyre$Analyte == "bifenthrin" &
                    Pyre$Unit == "ng/L"] /1000
      
      # Correct units
         Pyre$Unit[Pyre$Analyte == "bifenthrin" &
                      Pyre$Matrix == "water"] <- "ppb"

Pyre %>% filter(Analyte== "bifenthrin") %>% distinct(Unit)
```

#### esfenvalerate

In the analyte list, there are "esfenvalerate", "esfenvalerate/fenvalerate", "esfenvalerate/fenvalerate-1",
"esfenvalerate/fenvalerate-2"

Fenvalerate is a synthetic pyrethroid insecticide. It is a mixture of four optical isomers which have different insecticidal activities. 

Esfenvalerate, the 2-S alpha (or SS) configuration of fenvalerate, is the most insecticidally active isomer. Fenvalerate consists of about 23% of this isomer.

MM fenvalerate = MM esfenvalerate = 419.91 g/mol

For our purposes, it would be appropriate to rename all of these to esfenvalerate, since by definition the isomer always contains 23% of the fenvalerate isomer. 


```r
# esfenvalerate

  # Sediment:	ppb	and ng/g dw, and ug/kg	(equivalent units)
    Pyre$Unit[Pyre$Analyte == "esfenvalerate" &
                           Pyre$Matrix == "sediment"] <- "ppb"

  # Water: ng/L, ug/L, ppb
      # ppb = ng/L / 1000
      # ppb = ug/L
        
      # Convert ng/L to ppb
        Pyre$Result[Pyre$Analyte == "esfenvalerate" &
                  Pyre$Unit == "ng/L"] <- Pyre$Result[
                    Pyre$Analyte == "esfenvalerate" &
                    Pyre$Unit == "ng/L"] /1000
      
      # Correct units
         Pyre$Unit[Pyre$Analyte == "esfenvalerate" &
                      Pyre$Matrix == "water"] <- "ppb"

# esfenvalerate/fenvalerate

  # Sediment:	ng/g dw	and	ug/Kg dw (equivalent units)
    Pyre$Unit[Pyre$Analyte == "esfenvalerate/fenvalerate" &
                           Pyre$Matrix == "sediment"] <- "ppb"

  # Water: pg/L, ng/L, ug/L, ppb
      # ppb = pg/L /(1000*1000)
      # ppb = ng/L / 1000
      # ppb = ug/L
      
      # Convert pg/L to ppb
        Pyre$Result[Pyre$Analyte == "esfenvalerate/fenvalerate" &
              Pyre$Unit == "pg/L"] <- Pyre$Result[
                Pyre$Analyte == "esfenvalerate/fenvalerate" &
                Pyre$Unit == "pg/L"] /(1000*1000)
        
      # Convert ng/L to ppb
        Pyre$Result[Pyre$Analyte == "esfenvalerate/fenvalerate" &
                  Pyre$Unit == "ng/L"] <- Pyre$Result[
                    Pyre$Analyte == "esfenvalerate/fenvalerate" &
                    Pyre$Unit == "ng/L"] /1000
      
      # Correct units
         Pyre$Unit[Pyre$Analyte == "esfenvalerate/fenvalerate" &
                      Pyre$Matrix == "water"] <- "ppb"
         
# Correct units for others (sed samples, equivalent to ppb)
  Pyre$Unit[Pyre$Analyte == "esfenvalerate/fenvalerate-1"] <- "ppb"
  Pyre$Unit[Pyre$Analyte == "esfenvalerate/fenvalerate-2"] <- "ppb"

# Correct name
  
Pyre$Analyte[Pyre$Analyte == "esfenvalerate/fenvalerate"] <- "esfenvalerate"

Pyre$Analyte[Pyre$Analyte == "esfenvalerate/fenvalerate-1"] <- "esfenvalerate"

Pyre$Analyte[Pyre$Analyte == "esfenvalerate/fenvalerate-2"] <- "esfenvalerate"
```

#### **Pyre Result**

```r
Pyre %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```

```
## # A tibble: 6 x 5
## # Groups:   Analyte, Matrix [6]
##   Analyte       Matrix   Unit      n     mean
##   <chr>         <chr>    <chr> <int>    <dbl>
## 1 bifenthrin    sediment ppb     522 6.51    
## 2 bifenthrin    water    ppb    1018 0.00168 
## 3 cyfluthrin    sediment ppb     522 0.799   
## 4 cyfluthrin    water    ppb     915 0.000178
## 5 esfenvalerate sediment ppb     523 0.912   
## 6 esfenvalerate water    ppb     892 0.000121
```



```r
Wide.Pyre.Waterdf <- Pyre %>% filter(!Matrix == "sediment") %>%
  group_by(Date, Latitude, Longitude, Analyte, Matrix) %>%
  summarize(Subregion = first(Subregion),
            Mean = mean(Result, na.rm = T)) %>%
  pivot_wider(names_from = Analyte,
              names_repair = "check_unique",
              values_from = Mean) # Values to fill columns
```

```
## `summarise()` has grouped output by 'Date', 'Latitude', 'Longitude', 'Analyte'. You can override using the `.groups` argument.
```

```r
write.csv(x = Wide.Pyre.Waterdf , file = "Data/Output/WideSubsets/Pyre.Wide.water.csv", 
          row.names = F)
```
<br>

### GABA inhibitors  {.tabset}

AKA anything fipronil. Info on Fipronil and its degradates summarized from http://npic.orst.edu/factsheets/archive/fiptech.html


```r
GABA <- CEDENSURF %>% filter(SelectList == "GABA")

GABA %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```
**Fipronil** is a broad-spectrum insecticide that belongs to the phenylpyrazole chemical family. Fipronil disrupts the insect central nervous system.

**Fipronil-sulfone** is the primary biological metabolite of fipronil, is reported to be twenty times more active at mammalian chloride channels than at insect chloride channels. 10 Fipronil-sulfone is reportedly six times more potent in blocking vertebrate GABA-gated chloride channels than fipronil, but demonstrates similar toxicity to the parent compound in mammals.

**Fipronil-desulfinyl**, the primary environmental metabolite (photoproduct) of fipronil, is 9-10 times more active at the mammalian chloride channel than the parent compound, reducing the selectivity between insects and humans when exposed to this metabolite.

**Fipronil-amide** is another degradate of Fipronil (https://pubmed.ncbi.nlm.nih.gov/32574918/)


#### Constant units, no changes needed

fipronil detrifluoromethylsulfinyl - (sediment only) ug/Kg dw

#### fipronil

fipronil - ppb (sed and water)
fipronil amide


```r
# fipronil

  # Sediment:	ppb	and ng/g dw, and ug/kg	(equivalent units)
    GABA$Unit[GABA$Analyte == "fipronil" &
                           GABA$Matrix == "sediment"] <- "ppb"

  # Water: ng/L, ug/L, ppb
      # ppb = pg/L / (1000*1000)
      # ppb = ng/L / 1000
      # ppb = ug/L
        
      # Convert pg/L to ppb
        GABA$Result[GABA$Analyte == "fipronil" &
              GABA$Unit == "pg/L"] <- GABA$Result[
                GABA$Analyte == "fipronil" &
                GABA$Unit == "pg/L"] /(1000*1000)
        
      # Convert ng/L to ppb
        GABA$Result[GABA$Analyte == "fipronil" &
                  GABA$Unit == "ng/L"] <- GABA$Result[
                    GABA$Analyte == "fipronil" &
                    GABA$Unit == "ng/L"] /1000
      
      # Correct units
         GABA$Unit[GABA$Analyte == "fipronil" &
                      GABA$Matrix == "water"] <- "ppb"
```

fipronil amide - ppb


```r
# fipronil amide
  # Assess coverage
    GABA %>% filter(Analyte == "fipronil amide") %>%
      distinct(Subregion) # coverage in 2 regions

  # Sediment:	ppb	and ng/g dw, and ug/kg	(equivalent units)
    GABA$Unit[GABA$Analyte == "fipronil amide" &
                           GABA$Matrix == "sediment"] <- "ppb"

  # Water: ug/L and ppb (equivalent units)
    GABA$Unit[GABA$Analyte == "fipronil amide" &
                GABA$Matrix == "water"] <- "ppb"
    
  # Name fix
    GABA$Analyte[GABA$Analyte == "fipronil amide"] <- "fipronil_amide"
```

fipronil desulfinyl = desulfinyl fipronil

sediment = ppb
water = ppb


```r
# fipronil desulfinyl
  
  # correct reversed name
  GABA$Analyte[GABA$Analyte == "desulfinyl fipronil"] <- "fipronil desulfinyl"
  
  # Units?
  GABA %>% filter(Analyte == "fipronil desulfinyl") %>% distinct(Unit)
  
  # Sediment: ppb, ng/g dw, and ug/kg	(equivalent units to ppb)
  GABA$Unit[GABA$Analyte == "fipronil desulfinyl" &
                           GABA$Matrix == "sediment"] <- "ppb"
  
  # Water: pg/L, ng/L, ug/L, ppb
    
      # ppb = pg/L / (1000*1000)
      # ppb = ng/L / 1000
      # ppb = ug/L
    
      # Convert pg/L to ppb
        GABA$Result[GABA$Analyte == "fipronil desulfinyl" &
              GABA$Unit == "ng/L"] <- GABA$Result[
                GABA$Analyte == "fipronil desulfinyl" &
                GABA$Unit == "ng/L"]/(1000*1000)
        
      # Convert ng/L to ppb
        GABA$Result[GABA$Analyte == "fipronil desulfinyl" &
              GABA$Unit == "ng/L"] <- GABA$Result[
                GABA$Analyte == "fipronil desulfinyl" &
                GABA$Unit == "ng/L"]/1000
        
      # Correct units
        GABA$Unit[GABA$Analyte == "fipronil desulfinyl" &
                      GABA$Matrix == "water"] <- "ppb"
```

fipronil desulfinyl amide = desulfinyl fipronil amide

sediment = ppb
water = ppb


```r
# fipronil desulfinyl amide
      # correct reversed name
      GABA$Analyte[GABA$Analyte == "desulfinyl fipronil amide"] <- 
            "fipronil desulfinyl amide"
      
      # Units?
      GABA %>% filter(Analyte == "fipronil desulfinyl amide") %>%
        distinct(Unit, Matrix)
  
  # Water: pg/L, ng/L, ug/L, ppb
      # ppb = pg/L / (1000*1000)
      # ppb = ng/L / 1000
      # ppb = ug/LConvert ng/L to ppb (ug/L)
      
      GABA$Result[GABA$Analyte == "fipronil desulfinyl amide" &
                  GABA$Unit == "ng/L"] <- GABA$Result[
                    GABA$Analyte == "fipronil desulfinyl amide" &
                    GABA$Unit == "ng/L"]/1000
      
      # Correct units
         GABA$Unit[GABA$Analyte == "fipronil desulfinyl amide" &
                      GABA$Matrix == "water"] <- "ppb"
         
  # Sediment: ppb, ng/g dw (equivalent units)
        GABA$Unit[GABA$Analyte == "fipronil desulfinyl amide" &
                           GABA$Matrix == "sediment"] <- "ppb"
```

#### fipronil sulfide - ppb


```r
# fipronil sulfide

  # Sediment:	ppb	and ng/g dw, and ug/kg	(equivalent units)
    GABA$Unit[GABA$Analyte == "fipronil sulfide" &
                           GABA$Matrix == "sediment"] <- "ppb"

  # Water: ng/L, ug/L, ppb
      # ppb = pg/L / (1000*1000)
      # ppb = ng/L / 1000
      # ppb = ug/L
        
      # Convert pg/L to ppb
        GABA$Result[GABA$Analyte == "fipronil sulfide" &
              GABA$Unit == "pg/L"] <- GABA$Result[
                GABA$Analyte == "fipronil sulfide" &
                GABA$Unit == "pg/L"] /(1000*1000)
        
      # Convert ng/L to ppb
        GABA$Result[GABA$Analyte == "fipronil sulfide" &
                  GABA$Unit == "ng/L"] <- GABA$Result[
                    GABA$Analyte == "fipronil sulfide" &
                    GABA$Unit == "ng/L"] /1000
      
      # Correct units
         GABA$Unit[GABA$Analyte == "fipronil sulfide" &
                      GABA$Matrix == "water"] <- "ppb"
```

fipronil sulfone


```r
# fipronil sulfone

  # Sediment:	ppb	and ng/g dw, and ug/kg	(equivalent units)
    GABA$Unit[GABA$Analyte == "fipronil sulfone" &
                           GABA$Matrix == "sediment"] <- "ppb"

  # Water: ng/L, ug/L, ppb
      # ppb = pg/L / (1000*1000)
      # ppb = ng/L / 1000
      # ppb = ug/L
        
      # Convert pg/L to ppb
        GABA$Result[GABA$Analyte == "fipronil sulfone" &
              GABA$Unit == "pg/L"] <- GABA$Result[
                GABA$Analyte == "fipronil sulfone" &
                GABA$Unit == "pg/L"] /(1000*1000)
        
      # Convert ng/L to ppb
        GABA$Result[GABA$Analyte == "fipronil sulfone" &
                  GABA$Unit == "ng/L"] <- GABA$Result[
                    GABA$Analyte == "fipronil sulfone" &
                    GABA$Unit == "ng/L"] /1000
      
      # Correct units
         GABA$Unit[GABA$Analyte == "fipronil sulfone" &
                      GABA$Matrix == "water"] <- "ppb"
         
# Correct names

GABA$Analyte[GABA$Analyte == "fipronil desulfinyl"] <- 
  "fipronil_desulfinyl"

GABA$Analyte[GABA$Analyte == "fipronil desulfinyl amide"] <-
    "fipronil_desulfinyl_amide"

GABA$Analyte[GABA$Analyte == "fipronil sulfide"] <- "fipronil_sulfide"

GABA$Analyte[GABA$Analyte == "fipronil sulfone"] <- "fipronil_sulfone"

# Remove if insufficient replication
  GABA <- GABA %>% 
    filter(!Analyte == "fipronil detrifluoromethylsulfinyl") %>%
    filter(!Analyte == "fipronil_amide")
```

#### **GABA Result**

```r
GABA %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```

```
## # A tibble: 10 x 5
## # Groups:   Analyte, Matrix [10]
##    Analyte                   Matrix   Unit      n       mean
##    <chr>                     <chr>    <chr> <int>      <dbl>
##  1 fipronil                  sediment ppb     383  0.00253  
##  2 fipronil                  water    ppb     659  0.00114  
##  3 fipronil_desulfinyl       sediment ppb     396  0.0549   
##  4 fipronil_desulfinyl       water    ppb     649  0.000370 
##  5 fipronil_desulfinyl_amide sediment ppb     299 -0.00887  
##  6 fipronil_desulfinyl_amide water    ppb     522  0.0000759
##  7 fipronil_sulfide          sediment ppb     176  0.361    
##  8 fipronil_sulfide          water    ppb     244  0.000237 
##  9 fipronil_sulfone          sediment ppb     174  0.450    
## 10 fipronil_sulfone          water    ppb     321  0.00108
```



```r
Wide.GABA.Waterdf <- GABA %>% filter(!Matrix == "sediment") %>%
  group_by(Date, Latitude, Longitude, Analyte, Matrix) %>%
  summarize(Subregion = first(Subregion),
            Mean = mean(Result, na.rm = T)) %>%
  pivot_wider(names_from = Analyte,
              names_repair = "check_unique",
              values_from = Mean) # Values to fill columns
```

```
## `summarise()` has grouped output by 'Date', 'Latitude', 'Longitude', 'Analyte'. You can override using the `.groups` argument.
```

```r
write.csv(x = Wide.GABA.Waterdf , file = "Data/Output/WideSubsets/GABA.Wide.water.csv", 
          row.names = F)
```

<br>

### Others: Glyphosate, Atrazine, Neonicitinoids  {.tabset}


```r
Other <- CEDENSURF %>% filter(SelectList == c("Glyphosate", "Atrazine", "Neon"))
```

```
## Warning in SelectList == c("Glyphosate", "Atrazine", "Neon"): longer object
## length is not a multiple of shorter object length
```

#### Neonicotinoids 

All measures are in water. 

hydroxy-imidacloprid (2 records total)
imidacloprid (228 records)


```r
Other %>% filter(SelectList == "Neon") %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```


```r
  # Water: ng/L, ug/L, ppb

      # ppb = ng/L / 1000
      # ppb = ug/L

      # Convert ng/L to ppb
        Other$Result[Other$SelectList == "Neon" &
                  Other$Unit == "ng/L"] <- Other$Result[
                    Other$SelectList == "Neon" &
                    Other$Unit == "ng/L"] /1000
      
      # Correct units
         Other$Unit[Other$SelectList == "Neon"] <- "ppb"
         
# Remove records with insufficient replication
  Other <- Other %>% filter(!Analyte == "hydroxy-imidacloprid")
```

#### Glyphosate - ppb

(water only) 219 records, no conversion needed (equivalent units)


```r
Other %>% filter(SelectList == "Glyphosate") %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```

```r
# units ug/L and ppb (equivalent units)
  Other$Unit[Other$SelectList == "Glyphosate"] <- "ppb"
```

#### Atrazine

Atrazine (sed and water) - ppb
Atrazine degradate (water only) - ppb


```r
Other %>% filter(SelectList == "Atrazine") %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```

```r
# Water: ng/L, ug/L, ppb

      # ppb = ng/L / 1000
      # ppb = ug/L

      # Convert ng/L to ppb
        Other$Result[Other$SelectList == "Atrazine" &
                  Other$Unit == "ng/L"] <- Other$Result[
                    Other$SelectList == "Atrazine" &
                    Other$Unit == "ng/L"] /1000
      
      # Correct units
         Other$Unit[Other$SelectList == "Atrazine"] <- "ppb"
         
# Remove records with insufficient replication
  Other <- Other %>% filter(!Analyte == "atrazine degradate")
```

#### **Other Result**

```r
Other %>%
  group_by(Analyte, Matrix, Unit) %>%
  summarise(n = n(),
            mean = mean(Result))
```

```
## `summarise()` has grouped output by 'Analyte', 'Matrix'. You can override using the `.groups` argument.
```

```
## # A tibble: 4 x 5
## # Groups:   Analyte, Matrix [4]
##   Analyte      Matrix   Unit      n    mean
##   <chr>        <chr>    <chr> <int>   <dbl>
## 1 atrazine     sediment ppb      98 0      
## 2 atrazine     water    ppb     472 0.00472
## 3 glyphosate   water    ppb     232 0.829  
## 4 imidacloprid water    ppb     224 0.00233
```



```r
Wide.Other.Waterdf <- Other %>% filter(!Matrix == "sediment") %>%
  group_by(Date, Latitude, Longitude, Analyte, Matrix) %>%
  summarize(Subregion = first(Subregion),
            Mean = mean(Result, na.rm = T)) %>%
  pivot_wider(names_from = Analyte,
              names_repair = "check_unique",
              values_from = Mean) # Values to fill columns
```

```
## `summarise()` has grouped output by 'Date', 'Latitude', 'Longitude', 'Analyte'. You can override using the `.groups` argument.
```

```r
write.csv(x = Wide.Other.Waterdf , file = "Data/Output/WideSubsets/Other.Wide.water.csv", 
          row.names = F)
```

<br>


### Save

Compile and Save Modified Dataset (Long)

The file Data/Output/CEDENSURF_Limited_FixedUnits.csv contains all corrected units and analytes chosen for our model in long format. 


```r
Limited <- rbind(WQP, GABA, Metal, OrganoP, Pyre, Other)

write.csv(x = Limited, 
          file = "Data/Output/CEDENSURF_Limited_FixedUnits.csv", 
          na = "", row.names = F)
```

## 3. Summarize into wide format

I used pivot_wider to summarize analyte results by date and subregion within each matrix. 

Because column names are the means by which Netica differentiates nodes, I made the wide format data from the water-matrix contain just the analyte as the column name, while the sediment data includes analyte_sediment as the column names

Allwater.Wide.csv contains the final summarized analytes in wide format (one row per date/location; one column per analyte)

Allsed.Wide.csv contains the final summarized analytes in wide format (one row per date/location; one column per analyte)


```r
## Subset: Water Matrix

# Convert long to wide
Wide.waterdf <- Limited %>% filter(!Matrix == "sediment") %>%
  group_by(Date, Latitude, Longitude, Analyte, Matrix) %>%
  summarize(Subregion = first(Subregion),
            Mean = mean(Result, na.rm = T)) %>%
  pivot_wider(names_from = Analyte,
              names_repair = "check_unique",
              values_from = Mean) # Values to fill columns
```

```
## `summarise()` has grouped output by 'Date', 'Latitude', 'Longitude', 'Analyte'. You can override using the `.groups` argument.
```

```r
# save document
write.csv(x = Wide.waterdf, file = "Data/Output/Allwater.Wide.csv", 
          row.names = F)
```


```r
## Subset: Water Matrix

# Convert long to wide
Wide.seddf <- Limited %>% filter(!Matrix == "water") %>%
  group_by(Date, Latitude, Longitude, Analyte, Matrix) %>%
  summarize(Subregion = first(Subregion),
            Mean = mean(Result, na.rm = T)) %>%
  pivot_wider(names_from = c(Analyte,Matrix),
              names_repair = "check_unique",
              values_from = Mean) # Values to fill columns
```

```
## `summarise()` has grouped output by 'Date', 'Latitude', 'Longitude', 'Analyte'. You can override using the `.groups` argument.
```

```r
# save document
write.csv(x = Wide.seddf, file = "Data/Output/Allsed.Wide.csv", 
          row.names = F)
```