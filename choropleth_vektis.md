Choropleth with Vektis data
================
Hugo Janssen (nl-hugo)
16-10-2017

This document creates a basic choropleth from a CBS shapefile containing data about The Netherlands. Data provided by the Dutch Statistics Center [CBS](https://www.cbs.nl/en-gb), [Kadaster](https://www.kadaster.com/) and [Vektis](https://www.vektis.nl/streams/open-data).

### Setup

Load the required packages and specify the file locations.

``` r
#install.packages(c("rgeos", "maptools", "ggplot2", "plyr", "dplyr", "ggthemes"))
library(rgeos)
library(ggplot2)
library(ggthemes)
library(plyr)
library(dplyr)

# specify file locations
cbsURL <-"https://www.cbs.nl/-/media/_pdf/2017/36/buurt_2017.zip"
vektisURL <- "https://www.vektis.nl/uploads/Docs%20per%20pagina/Open%20Data%20Bestanden/Vektis%20Open%20Databestand%20Zorgverzekeringswet%202015%20-%20gemeente.csv"
datadir <- "data"
```

### Download

Download the zipped data file and unzip. Note that files are downloaded only when no data folder is present.

``` r
csv <- paste(datadir, "gemeente_2015.csv", sep = "/")

# create a dir and download if no datadir is present 
# assumes that files are downloaded if a datadir is present
if (!file.exists(datadir)) {
  dir.create(datadir)

  # cbs bestanden
  tmpfile <- paste(datadir, basename(cbsURL), sep = "/")
  download.file(cbsURL, destfile = tmpfile)
  unzip(tmpfile, exdir="./data")
  
  # vektis bestanden
  download.file(vektisURL, destfile = csv)
}
```

### Prepare data

Read Vektis data first and aggregate by municipality. Municipality names that differ from the official CBS name, need to be renamed.

``` r
# read csv
vektis.df <- read.csv(csv, sep = ";", stringsAsFactors = FALSE)
names(vektis.df) <- tolower(names(vektis.df))

# compute cost per insured person
vektis.df$gem_kosten <- vektis.df$kosten_medisch_specialistische_zorg / vektis.df$aantal_verzekerdejaren

# aggregate
gem.df <- aggregate(gem_kosten ~ gemeentenaam, vektis.df, sum)

# Vektis 2015 uses 2017 CBS names
# fix broken names before joining the dataframes
gm.vektis <- c(
  "S GRAVENHAGE",
  "S HERTOGENBOSCH",
  "BERGEN LB",
  "BERGEN NH",
  "HAARLEMMERLIEDE CA",
  "KOLLUMERLAND CA",
  "NOORD BEVELAND",
  "NUENEN CA",
  "SUDWEST-FRYSLAN"
  )
  
gm.cbs <- c(
  "'S-GRAVENHAGE",
  "'S-HERTOGENBOSCH",
  "BERGEN (L.)",
  "BERGEN (NH.)",
  "HAARLEMMERLIEDE EN SPAARNWOUDE",
  "KOLLUMERLAND EN NIEUWKRUISLAND",
  "NOORD-BEVELAND",
  "NUENEN, GERWEN EN NEDERWETTEN",
  "SÃºDWEST-FRYSLÃ¢N"
  )
  
gem.df$gemeentenaam <- mapvalues(gem.df$gemeentenaam, from = gm.vektis, to = gm.cbs)
```

Read the topology from the shapefile and turn it into a dataframe that can be plotted by `ggplot`. Note that areas that are marked as sea or lake (by the `WATER` property) are excluded, so that coast is displayed nicely.

``` r
# prepare filenames
basename <- gsub("buurt", "gem", tools::file_path_sans_ext(basename(cbsURL)))
shpfile <- paste(basename, "shp", sep = ".")

# create spatial object
nl <- maptools::readShapeSpatial(paste(datadir, shpfile, sep = "/"))

# remove watery areas
nl <- nl[nl@data$WATER == "NEE", ]

# create id to join data
nl@data$id <- as.character(nl@data$GM_CODE)

# convert to ggplot object
nl.points <- fortify(nl, region = "GM_CODE")

# join the geo points and the data
nl.df <- nl.points %>% 
  left_join(nl@data, by = "id") %>% 
  mutate(GM_NAAM = toupper(as.character(GM_NAAM))) %>%
  left_join(gem.df, by = c("GM_NAAM" = "gemeentenaam"))
```

### Plot

Create a basic plot of the map with minimal styling.

``` r
# create the plot
p <- ggplot(nl.df) + 
  aes(x = long, y = lat, group = group, fill = gem_kosten) + 
  geom_polygon() +
  geom_path(colour = "white", size = 0.25) +
  scale_fill_distiller(palette = "PiYG") + 
  labs(x = NULL, 
       y = NULL, 
       title = "Kosten MSZ",
       subtitle = "Gemiddelde kosten per verzekerde (2015)") +
  coord_equal() +
  theme_minimal() +
  theme (
    panel.grid.major = element_blank(), 
    panel.grid.minor = element_blank(),
    axis.text = element_blank()
  )

p
```

<img src="img/kosten-msz-choropleth-1.png" width="100%" />
