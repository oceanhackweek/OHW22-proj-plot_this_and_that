Accessing and plotting FishMIP data
================
Denisse Fierro Arcos
2022-08-17

-   [Introduction](#introduction)
    -   [Setting up the notebook](#setting-up-the-notebook)
        -   [Loading R libraries](#loading-r-libraries)
        -   [Setting up Python for use in an R
            notebook](#setting-up-python-for-use-in-an-r-notebook)
    -   [Python-based code](#python-based-code)
        -   [Loading ISIMIP Client
            script](#loading-isimip-client-script)
        -   [Search ISIMIP repository](#search-isimip-repository)
        -   [Downloading data to disk](#downloading-data-to-disk)
    -   [R-based code](#r-based-code)
        -   [Inspecting contents of netcdf
            file](#inspecting-contents-of-netcdf-file)
        -   [Loading dataset as dataframe for easy
            manipulation](#loading-dataset-as-dataframe-for-easy-manipulation)
        -   [Calculating climatology](#calculating-climatology)
        -   [Plotting climatology](#plotting-climatology)
        -   [Obtaining timeseries](#obtaining-timeseries)

# Introduction

This notebook will accessing data from the [Inter-Sectoral Impact Model
Intercomparison Project (ISIMIP) Repository](https://data.isimip.org/).
There are various datasets available, but as example we will be
accessing Output Data \> Fisheries & Marine Ecosystems (global) sector
\> ISIMIP3b simulation.

In this notebook, we will use `Python` to make use of some functions
from the `isimip-client` library. We will then move to `R` to manipulate
the data and produce maps.

## Setting up the notebook

### Loading R libraries

``` r
library(reticulate)
```

    ## Warning: package 'reticulate' was built under R version 4.2.1

``` r
library(tidyverse)
library(metR)
```

    ## Warning: package 'metR' was built under R version 4.2.1

``` r
library(lubridate)
library(raster)
library(sf)
```

### Setting up Python for use in an R notebook

Using the `reticulate` package. You have the option to set the path of
Python installation using
`reticulate::use_python("PATH_TO_PYTHON_INSTALLATION")`, or to point to
a specific conda environment using
`reticulate::use_condaenv("conda_env")`.

#### Search Python installation path

You can use the `Sys.which()` function in `R` to search the path of the
Python installation in your machine using the command below. We will
save the path to a variable so we can use it while setting up `Python`
access in this notebook.

``` r
python_path <- as.character(Sys.which("python"))
```

We can now use the above variable when calling `Python` to this
notebook. But as explained above, you can also call a specific
environment.

``` r
#Point to location of Python installation in your machine
use_python(python_path)

#Alternatively, point to a specific conda environment
#use_condaenv("/opt/conda")
```

## Python-based code

### Loading ISIMIP Client script

The script originally came from the ISIMIP-Client Python package. Their
source code is available online in their
[repository](https://github.com/ISI-MIP/isimip-client). We will now load
the script containing these functions.

Alternatively, you can install the entire library using the following
line of code:
`!pip install git+https://github.com/ISI-MIP/isimip-client` in a
`Python` chunk.

``` python
#Calling the script with the ISIMIP-client functions
import client as cl

#Starting a session
client = cl.ISIMIPClient()
```

### Search ISIMIP repository

You can create a query specifying several attributes of the dataset of
your interest. In this case, we are interested in Total Catch (`tc` for
short), which is a type of Output Data from the ISIMIP3b simulation. For
this example, we have also specified an ecological model (`EcoOcean`),
the climate forcing used (`GFDL-ESM4` model) and the climate scenario
(`historical`).

``` python
query = client.datasets(simulation_round = 'ISIMIP3b',\
                           product = 'OutputData',\
                           climate_forcing = 'gfdl-esm4',\
                           climate_scenario = 'historical',\
                           model = 'ecoocean',\
                           variable = 'tc')
```

This query will return a Python dictionary with the results of your
search. You can find the name of the keys in the dictionary by typing
`query.keys()`. You can inspect the search results by typing
`query['name_of_key"]`.

For example, the `query['count']` contains the total number of datasets
which met your search requirements. In this case, we found 1 datasets
meeting our search requirements.

We can also check the additional metadata associated stored in
`query['results']` as shown below.

``` python
#We will loop through all search results available
for dataset in query['results']:
  #Print metadata for each entry
  print(dataset['specifiers'])
```

    ## {'model': 'ecoocean', 'period': 'historical', 'region': 'global', 'sector': 'marine-fishery_global', 'product': 'OutputData', 'variable': 'tc', 'time_step': 'monthly', 'soc_scenario': 'histsoc', 'sens_scenario': 'default', 'bias_adjustment': 'nobasd', 'climate_forcing': 'gfdl-esm4', 'climate_scenario': 'historical', 'simulation_round': 'ISIMIP3b'}

It is worth noting that the files in the search results include data for
the entire planet. If you would like to

can be downloaded to disk by extracting the URLs from the dictionary. If
a subset of the global data is needed, then the paths information is
necessary. Here we will save both options.

``` python
#Empty lists to save URLs linking to files
urls = []
urls_sub = []

#Looping through each entry available in search results
for datasets in query['results']:
  for paths in dataset['files']:
    urls.append(paths['file_url'])
    urls_sub.append(paths['path'])
```

#### Optional step

If you are looking for a subset of the global dataset, you can set a
bounding box and only extract data for your area of interest. In this
case, we will extract information for the Southern Ocean only.

``` python
#We use the cutout function to create a bounding box for our dataset
SO_data_URL = client.cutout(urls_sub, bbox=[-90, -40, -180, 180])
```

### Downloading data to disk

We will download the data and store it into the `data` folder. First we
will make sure a `data` folder exists and if it does not exist, we will
create one.

``` python
#Importing library to check if folder exists
import os

#Creating a data folder if one does not already exist
if os.path.exists('../data/') == False:
  os.makedirs('../data/')
else:
  print('Folder already exists')
```

    ## Folder already exists

Use the `client.download()` function to save data to disk.

``` python
#To download the subsetted data
client.download(url = SO_data_URL['file_url'], \
                path = '../data/', validate = False, \
                extract = True)

#To download the global data
# client.download(url = urls[0], \
#                 path = '../data/', validate = False, \
#                 extract = True)
```

## R-based code

You are now ready to load the dataset into `R` to make any calculations
and visualise results.

### Inspecting contents of netcdf file

For a quick overview of the content of the dataset we just downloaded,
we can make use of the `metR` package.

``` r
#Provide file path to netcdf that was recently downloaded.
data_file <- "../data/ecoocean_gfdl-esm4_nobasd_historical_histsoc_default_tc_lat-90.0to-40.0lon-180.0to180.0_monthly_1950_2014.nc"

#Check contents of netcdf
GlanceNetCDF(data_file)
```

    ## ----- Variables ----- 
    ## tc:
    ##     Total Catch (all commercial functional groups / size classes) in g m-2
    ##     Dimensions: lon by lat by time
    ## 
    ## 
    ## ----- Dimensions ----- 
    ##   lat: 50 values from -89.5 to -40.5 degrees_north
    ##   lon: 360 values from -179.5 to 179.5 degrees_east
    ##   time: 780 values from 1955-01-30 12:39:26 to 2020-12-10 20:00:09

This output, however, does not give you information about
`No Data Values`. So we will load the first timestep included in our
dataset to obtain this information.

We will also plot the data to inspect it quickly.

``` r
#Loading the first timestep as raster
tc_raster <- raster(data_file, band = 1)

#Extracting missing values
NA_val <- tc_raster@file@nodatavalue

#Plotting raster
plot(tc_raster)
```

![](Plotting_FishMIP_data_files/figure-gfm/unnamed-chunk-12-1.png)<!-- -->
We can see that a `No Data Values` is included in the dataset to mark
areas where no values have been collected because all land areas have
the same value.

We can create a mask for the `No Data Values` and plot the raster again.

``` r
#Changing values larger than No Data Value to NA
tc_raster[tc_raster >= NA_val] <- NA

#Plotting result
plot(tc_raster)
```

![](Plotting_FishMIP_data_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

### Loading dataset as dataframe for easy manipulation

This data type allows us to use the `tidyverse` to make calculations
easily.

As seen from plotting the raster above, we must first mask
`No Data Values` before we carry out any calculations.

``` r
#Loading dataset as data frame
tc_SO <- ReadNetCDF(data_file, vars = "tc") %>% 
  #Masking No Data Values
  mutate(tc = case_when(tc >= NA_val ~ NA_real_,
         T ~ tc))
```

    ## Warning in as.POSIXct.PCICt(PCICt::as.PCICt(time, cal = calendar, origin =
    ## origin), : 360-day PCICt objects can't be properly represented by a POSIXct
    ## object

### Calculating climatology

We will use all data between 1960 and 2020 to calculate the
climatological mean of total catch for the Southern Ocean.

``` r
clim_tc <- tc_SO %>% 
  #Extracting data between 1960 and 2020
  filter(year(time) >= 1960 & year(time) <= 2020) %>% 
  #Calculating climatological mean for total catch per pixel
  group_by(lat, lon) %>% 
  summarise(mean_tc = mean(tc, na.rm = F))
```

    ## `summarise()` has grouped output by 'lat'. You can override using the `.groups` argument.

### Plotting climatology

``` r
#Accessing coastline shapefiles
land <- rnaturalearth::ne_countries(returnclass = "sf") %>% 
  #Extracting land within data boundaries
  st_crop(c(ymin = -90, ymax = -40, 
            xmin = -180, xmax = 180)) 
```

    ## although coordinates are longitude/latitude, st_intersection assumes that they are planar

    ## Warning: attribute variables are assumed to be spatially constant throughout all
    ## geometries

``` r
#Plotting data
clim_tc %>% 
  ggplot(aes(y = lat, x = lon))+
  geom_contour_fill(aes(z = mean_tc, fill = stat(level)),
                        binwidth = 2)+
  scale_fill_distiller(palette = "YlOrBr", 
                       direction = 1, 
                       super = ScaleDiscretised)+
  guides(fill = guide_colorbar(title.position = "top",
                               title.hjust =  0.5,
                               barwidth = 15))+
  geom_sf(data = land, inherit.aes = F,
          fill = "grey", color = "black")+
  labs(x = NULL, y = NULL, 
       fill = "Total Catch in g m-2")+
  theme_bw()+
  theme(panel.grid = element_blank(),
        legend.position = "bottom")
```

![](Plotting_FishMIP_data_files/figure-gfm/unnamed-chunk-16-1.png)<!-- -->

### Obtaining timeseries

``` r
# tc_SO %>% 
#   #Calculating climatological mean for total catch per pixel
#   group_by(time) %>% 
#   summarise(mean_tc = mean(tc, na.rm = F)) %>% View()
#   ggplot() %>% 
```
