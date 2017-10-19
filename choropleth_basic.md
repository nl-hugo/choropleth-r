Basic choropleth
================
Hugo Janssen (nl-hugo)
19-10-2017

This document creates a basic choropleth from a CBS shapefile containing data about The Netherlands. Data provided by the Dutch Statistics Center [CBS](https://www.cbs.nl/en-gb) and [Kadaster](https://www.kadaster.com/).

### Setup

Load the required packages and specify the file locations.

``` r
#install.packages(c("ggplot2", "dplyr", "ggthemes", "rgdal", "sp"))
library(ggplot2)
library(ggthemes)
library(dplyr)

# specify file locations
URL <-"https://www.cbs.nl/-/media/_pdf/2017/36/buurt_2015.zip"
datadir <- "data"
```

### Download

Download the zipped data file and unzip. Note that files are downloaded only not present in the data folder.

``` r
# create a dir if no datadir is present 
if (!file.exists(datadir)) {
  dir.create(datadir)
}

# download only when file does not exists
tmpfile <- paste(datadir, basename(URL), sep = "/")
if (!file.exists(tmpfile)) {
  download.file(URL, destfile = tmpfile)
  unzip(tmpfile, exdir = datadir)
}
```

### Prepare data

Read the topology from the shapefile and turn it into a dataframe that can be plotted by `ggplot`. Note that areas that are marked as sea or lake (by the `WATER` property) are excluded, so that coast is displayed nicely.

``` r
# define the layer to extract
layer <-
  gsub("buurt", "gem", tools::file_path_sans_ext(basename(URL)))
  
# read spatial object from file
nl <-
  rgdal::readOGR(
  datadir,
  layer = layer,
  verbose = FALSE,
  stringsAsFactors = FALSE
  )
  
# remove watery areas
nl <- nl[nl@data$WATER == "NEE",]

# reproject to WGS84
nl <-
  sp::spTransform(nl, sp::CRS("+proj=longlat +ellps=WGS84 +datum=WGS84"))
  
# create id to join data
nl@data$id <- as.character(nl@data$GM_CODE)

# convert to ggplot object
nl.points <- fortify(nl, region = "GM_CODE")

# join the geo points and the data
nl.df <- nl.points %>% left_join(nl@data, by = "id")
```

### Plot

Create a basic plot of the map with minimal styling.

``` r
# create the plot
p <- ggplot(nl.df) + 
  aes(x = long, y = lat, group = group, fill = as.numeric(P_GEBOO)) + 
  geom_polygon() +
  geom_path(colour = "white", size = 0.25) +
  scale_fill_distiller(palette = "PiYG") + 
  labs(x = NULL, 
       y = NULL, 
       title = "Geboortecijfer per gemeente",
       subtitle = "Aantal geboortes per 1000 inwoners") +
  coord_fixed(ratio = 1.5) +
  theme_minimal() +
  theme (
    panel.grid.major = element_blank(), 
    panel.grid.minor = element_blank(),
    axis.text = element_blank()
  )

p
```

<img src="img/basic-choropleth-1.png" width="100%" />
