# Reprojecting geographic data {#reproj-geo-data}

## Prerequisites {-}

- This chapter requires the following packages (**lwgeom** is also used, but does not need to be attached):


```r
library(sf)
library(terra)
library(dplyr)
library(spData)
library(spDataLarge)
```

## Introduction

Section \@ref(crs-intro) introduced coordinate reference systems (CRSs) and demonstrated their importance.
This chapter goes further, highlighting issues that can arise due to ignoring CRSs and to *transform* geographic data from one CRS to another.
\index{CRS!geographic} 
\index{CRS!projected} 
As illustrated in Figure \@ref(fig:vectorplots) from that earlier chapter, there are two types of CRSs: *geographic* ('lon/lat', with units in degrees longitude and latitude) and *projected* (typically with units of meters from a datum).
This has consequences.
To check if our data has geographic CRS, we can use `sf::st_is_longlat()` for vector data and `terra::is.lonlat()` for raster data.
In some cases the CRS is unknown, as is the case in the `london` dataset created in the code chunk below, building on the example of London introduced in Section \@ref(vector-data):


```r
london = data.frame(lon = -0.1, lat = 51.5) %>% 
  st_as_sf(coords = c("lon", "lat"))
st_is_longlat(london)
#> [1] NA
```

The output `NA` shows that `sf` does not know what the CRS is and is unwilling to guess (`NA` literally means 'not available').
Unless a CRS is manually specified or is loaded from a source that has CRS metadata, `sf` does not make any explicit assumptions about which coordinate systems, other than to say "I don't know".
This behavior makes sense given the diversity of available CRSs but differs from some approaches, such as the GeoJSON file format specification, which makes the simplifying assumption that all coordinates have a lon/lat CRS: `EPSG:4326`.
A CRS can be added to `sf` objects in three main ways:

- By assigning the CRS to a pre-existing object, e.g. with `st_crs(london) = "EPSG:4326"`.
- By passing a CRS to the `crs` argument in `sf` functions that create geometry objects such as `st_as_sf(... crs = "EPSG:4326")`. The same argument can also be used to set the CRS when creating raster datasets (e.g., `rast(crs = "EPSG:4326")`).
- With the `st_set_crs()`, which returns a version of the data that has a new CRS, an approach that is demonstrated in the following code chunk.


```r
london_geo = st_set_crs(london, "EPSG:4326")
st_is_longlat(london_geo)
#> [1] TRUE
```

Datasets without a specified CRS can cause problems: all geographic coordinates have a coordinate system and software can only make good decisions around plotting and and geometry operations if it knows what type of CRS it is working with.
If no CRS has been set, `sf` uses the GEOS geometry library for many operations.
GEOS is not well suited to lon/lat CRSs, as we will see later in this chapter.
If a CRS has been set, `sf` will use either GEOS or the S2 *spherical geometry engine* depending on the type of CRS.
<!-- Todo: add s2 section -->
<!--jn: s2 section is still missing from the book-->
Since `sf` version 1.0.0, R's ability to work with geographic vector datasets that have lon/lat CRSs has improved substantially, thanks to its integration with S2 introduced in Section \@ref(s2).
However, CRSs and transforming between them are still important, especially for geographic raster datasets for which spherical geometry is less relevant (S2 only works with vector geometries).
In this section we will demonstrate the importance of CRSs, and the impacts of using the S2 library for vector data, before moving on to the question of when to reproject in Section \@ref(whenproject) and techniques for reprojecting vector and raster objects in the remainder of the chapter. 

The example used in this introductory section is to create a buffer of 100 km around `london`.
We will also create a deliberately faulty buffer with a 'distance' of 1 degree, which is roughly equivalent to 100 km (1 degree is about 111 km at the equator).
Before diving into the code, it may be worth skipping briefly ahead to peek at Figure \@ref(fig:crs-buf) to get a visual handle on the outputs that you should be able to reproduce by following the code chunks below.

The first stage is to create three buffers around the `london` and `london_geo` objects created above with boundary distances of 1 degree and 100 km  (or 100,000 m, which can be expressed as `1e5` in scientific notation) from central London:


```r
london_buff_no_crs = st_buffer(london, dist = 1)   # incorrect: no CRS
london_buff_s2 = st_buffer(london_geo, dist = 1e5) # silent use of s2
london_buff_s2_100_cells = st_buffer(london_geo, dist = 1e5, max_cells = 100) 
```

In the first line above, `sf` assumes that the input is projected and generates a result that has a buffer in units of degrees, which is problematic, as we will see.
In the second line, `sf` silently uses the spherical geometry engine S2, introduced in Chapter \@ref(spatial-class), to calculate the extent of the buffer using the default value of `max_cells = 1000` --- set to `100` in line three --- the consequences which will become apparent shortly (see `?s2::s2_buffer_cells` for details).
To highlight the impact of `sf`'s use of the S2 geometry engine for unprojected (geographic) coordinate systems, we will temporarily disable it with the command `sf_use_s2()` (which is on, `TRUE`, by default), in the code chunk below.
Like `london_buff_no_crs`, the new `london_geo` object is a geographic abomination: it has units of degrees, which makes no sense in the vast majority of cases:


```r
sf::sf_use_s2(FALSE)
#> Spherical geometry (s2) switched off
london_buff_lonlat = st_buffer(london_geo, dist = 1) # incorrect result
#> Warning in st_buffer.sfc(st_geometry(x), dist, nQuadSegs, endCapStyle =
#> endCapStyle, : st_buffer does not correctly buffer longitude/latitude data
#> dist is assumed to be in decimal degrees (arc_degrees).
sf::sf_use_s2(TRUE)
#> Spherical geometry (s2) switched on
```

The results of the above code chunk show that, when spherical geometry operations are turned off, performing buffers (and other geometric operations) on unprojected datasets generate an important warning: the result of this operation may be of limited use because it is in units of latitude and longitude, rather than meters or some other suitable measure of distance.
<!--toDo:rl-->
<!--jn: something is wrong with the next sentence... -->
the buffer is elongated in the north-south direction because lines of longitude converge towards the Earth's poles.

\BeginKnitrBlock{rmdnote}<div class="rmdnote">The distance between two lines of longitude, called meridians, is around 111 km at the equator (execute `geosphere::distGeo(c(0, 0), c(1, 0))` to find the precise distance).
This shrinks to zero at the poles.
At the latitude of London, for example, meridians are less than 70 km apart (challenge: execute code that verifies this).
<!-- `geosphere::distGeo(c(0, 51.5), c(1, 51.5))` -->
Lines of latitude, by contrast, are equidistant from each other irrespective of latitude: they are always around 111 km apart, including at the equator and near the poles (see Figures \@ref(fig:crs-buf) to \@ref(fig:wintriproj)).</div>\EndKnitrBlock{rmdnote}

Do not interpret the warning about the geographic (`longitude/latitude`) CRS as "the CRS should not be set": it almost always should be!
It is better understood as a suggestion to *reproject* the data onto a projected CRS.
This suggestion does not always need to be heeded: performing spatial and geometric operations makes little or no difference in some cases (e.g., spatial subsetting).
But for operations involving distances such as buffering, the only way to ensure a good result (without using spherical geometry engines) is to create a projected copy of the data and run the operation on that.
<!--toDo:rl-->
<!-- jn: idea -- maybe it would be add a table somewhere in the book showing which operations are impacted by s2? -->
This is done in the code chunk below:


```r
london_proj = data.frame(x = 530000, y = 180000) %>% 
  st_as_sf(coords = 1:2, crs = "EPSG:27700")
```

The result is a new object that is identical to `london`, but reprojected onto a suitable CRS (the British National Grid, which has an EPSG code of 27700 in this case) that has units of meters.
We can verify that the CRS has changed using `st_crs()` as follows (some of the output has been replaced by `...`):



<!--toDo:rl-->
<!-- jn: the next paragraph need to be updated! -->
Notable components of this CRS description include the EPSG code (`EPSG: 27700`), the projection ([transverse Mercator](https://en.wikipedia.org/wiki/Transverse_Mercator_projection), `+proj=tmerc`), the origin (`+lat_0=49 +lon_0=-2`) and units (`+units=m`).^[
For a short description of the most relevant projection parameters and related concepts, see the fourth lecture by Jochen Albrecht hosted at
http://www.geography.hunter.cuny.edu/~jochen/GTECH361/lectures/ and information at https://proj.org/usage/projections.html.
Other great resources on projections are spatialreference.org and progonos.com/furuti/MapProj.
]
The fact that the units of the CRS are meters (rather than degrees) tells us that this is a projected CRS: `st_is_longlat(london_proj)` now returns `FALSE` and geometry operations on `london_proj` will work without a warning, meaning buffers can be produced from it using proper units of distance.
The following line of code creates a buffer around *projected* data of exactly 100 km:


```r
london_buff_projected = st_buffer(london_proj, 1e5)
```

The geometries of the three `london_buff*` objects that *have* a specified CRS created above (`london_buff_s2`, `london_buff_lonlat` and `london_buff_projected`) created in the preceding code chunks are illustrated in Figure \@ref(fig:crs-buf).



<div class="figure" style="text-align: center">
<img src="07-reproj_files/figure-html/crs-buf-1.png" alt="Buffers around London showing results created with the S2 spherical geometry engine on lon/lat data (left), projected data (middle) and lon/lat data without using spherical geometry (right). The left plot is the result of buffering lon/lat data with the default settings in sf which calls S2 spherical geometry engine by default and sets `max_cells` to 1000 (thin line) and with `max_cells` set to 100 (thick line). The gray outline represents the UK coastline." width="100%" />
<p class="caption">(\#fig:crs-buf)Buffers around London showing results created with the S2 spherical geometry engine on lon/lat data (left), projected data (middle) and lon/lat data without using spherical geometry (right). The left plot is the result of buffering lon/lat data with the default settings in sf which calls S2 spherical geometry engine by default and sets `max_cells` to 1000 (thin line) and with `max_cells` set to 100 (thick line). The gray outline represents the UK coastline.</p>
</div>

It is clear from Figure \@ref(fig:crs-buf) that buffers based on `s2` and properly projected CRSs are not 'squashed', meaning that every part of the buffer boundary is equidistant to London.
The results that are generated from lon/lat CRSs when `s2` is *not* used, either because the input lacks a CRS or because `sf_use_s2()` is turned off, are heavily distorted, with the result elongated in the north-south axis, highlighting the dangers of using algorithms that assume projected data on lon/lat inputs (as GEOS does).
The results generated using S2 are also distorted, however, although less dramatically.
Both buffer boundaries in Figure \@ref(fig:crs-buf) (left) are jagged, although this may only be apparent or relevant when for the thick boundary representing a buffer created with the `s2` argument `max_cells` set to 100.
<!--toDo:rl-->
<!--jn: maybe it is worth to emphasize that the differences are due to the use of S2 vs GEOS-->
<!--jn: you mention S2 a lot in this section, but not GEOS...-->
The less is that results obtained from lon/lat data via `s2` will be different from results obtained from using projected data, although these differences reduce as the value of `max_cells` increases: the 'right' value for this argument may depend on many factors and the default value 1000 is a reasonable default, balancing speed of computation against resolution of results, in many cases.
In situations where curved boundaries are advantageous, transforming to a projected CRS before buffering (or performing other geometry operations) may be appropriate.

The importance of CRSs (primarily whether they are projected or geographic) and the impacts of `sf`'s default setting to use S2 for buffers on lon/lat data is clear from the example above.
The subsequent sections go into more depth, exploring which CRS to use when projected CRSs *are* needed and the details of reprojecting vector and raster objects.

## When to reproject? {#whenproject}

\index{CRS!reprojection} 
The previous section showed how to set the CRS manually, with `st_set_crs(london, "EPSG:4326")`.
In real world applications, however, CRSs are usually set automatically when data is read-in.
In many projects the main CRS-related task is to *transform* objects, from one CRS into another.
But when should data be transformed? 
And into which CRS?
There are no clear-cut answers to these questions and CRS selection always involves trade-offs [@maling_coordinate_1992].
However, there are some general principles provided in this section that can help you decide. 

First it's worth considering *when to transform*.
<!--toDo:rl-->
<!--not longer valid-->
In some cases transformation to a projected CRS is essential, such as when using geometric functions such as `st_buffer()`, as Figure \@ref(fig:crs-buf) showed.
Conversely, publishing data online with the **leaflet** package may require a geographic CRS.
Another case is when two objects with different CRSs must be compared or combined, as shown when we try to find the distance between two objects with different CRSs:


```r
st_distance(london_geo, london_proj)
# > Error: st_crs(x) == st_crs(y) is not TRUE
```

To make the `london` and `london_proj` objects geographically comparable one of them must be transformed into the CRS of the other.
But which CRS to use?
The answer is usually 'the projected CRS', which in this case is the British National Grid (EPSG:27700):


```r
london2 = st_transform(london_geo, "EPSG:27700")
```

Now that a transformed version of `london` has been created, using the **sf** function `st_transform()`, the distance between the two representations of London can be found.
It may come as a surprise that `london` and `london2` are just over 2 km apart!^[
The difference in location between the two points is not due to imperfections in the transforming operation (which is in fact very accurate) but the low precision of the manually-created coordinates that created `london` and `london_proj`.
Also surprising may be that the result is provided in a matrix with units of meters.
This is because `st_distance()` can provide distances between many features and because the CRS has units of meters.
Use `as.numeric()` to coerce the result into a regular number.
]


```r
st_distance(london2, london_proj)
#> Units: [m]
#>      [,1]
#> [1,] 2018
```

## Which CRS to use?

<!--jn:toDo-->
<!--mention websites and the crssuggest package-->
<!-- https://epsg.org/home.html -->

<!--     Custom CRSs are also ideally specified as WKT2 -->
<!--     https://epsg.io/ -->
<!-- the two below websites are not up-to-date -->
<!--     https://spatialreference.org/ref/epsg/ -->
<!--     https://epsg.org/home.html -->

\index{CRS!reprojection} 
\index{projection!World Geodetic System}
The question of *which CRS* is tricky, and there is rarely a 'right' answer:
"There exist no all-purpose projections, all involve distortion when far from the center of the specified frame" [@bivand_applied_2013].

For **geographic CRSs**, the answer is often [WGS84](https://en.wikipedia.org/wiki/World_Geodetic_System#A_new_World_Geodetic_System:_WGS_84), not only for web mapping, but also because GPS datasets and thousands of raster and vector datasets are provided in this CRS by default.
WGS84 is the most common CRS in the world, so it is worth knowing its EPSG code: 4326.
This 'magic number' can be used to convert objects with unusual projected CRSs into something that is widely understood.

What about when a **projected CRS** is required?
In some cases, it is not something that we are free to decide:
"often the choice of projection is made by a public mapping agency" [@bivand_applied_2013].
This means that when working with local data sources, it is likely preferable to work with the CRS in which the data was provided, to ensure compatibility, even if the official CRS is not the most accurate.
The example of London was easy to answer because (a) the British National Grid (with its associated EPSG code 27700) is well known and (b) the original dataset (`london`) already had that CRS.

In cases where an appropriate CRS is not immediately clear, the choice of CRS should depend on the properties that are most important to preserve in the subsequent maps and analysis.
All CRSs are either equal-area, equidistant, conformal (with shapes remaining unchanged), or some combination of compromises of those (section \@ref(projected-coordinate-reference-systems)).
Custom CRSs with local parameters can be created for a region of interest and multiple CRSs can be used in projects when no single CRS suits all tasks.
'Geodesic calculations' can provide a fall-back if no CRSs are appropriate (see [proj.org/geodesic.html](https://proj.org/geodesic.html)).
Regardless of the projected CRS used, the results may not be accurate for geometries covering hundreds of kilometers.

When deciding on a custom CRS, we recommend the following:^[
<!--toDo:rl-->
<!-- jn:I we can assume who is the "anonymous reviewer", can we ask him/her to use his/her name? -->
Many thanks to an anonymous reviewer whose comments formed the basis of this advice.
]

\index{projection!Lambert azimuthal equal-area}
\index{projection!Azimuthal equidistant}
\index{projection!Lambert conformal conic}
\index{projection!Stereographic}
\index{projection!Universal Transverse Mercator}

- A Lambert azimuthal equal-area ([LAEA](https://en.wikipedia.org/wiki/Lambert_azimuthal_equal-area_projection)) projection for a custom local projection (set `lon_0` and `lat_0` to the center of the study area), which is an equal-area projection at all locations but distorts shapes beyond thousands of kilometers
- Azimuthal equidistant ([AEQD](https://en.wikipedia.org/wiki/Azimuthal_equidistant_projection)) projections for a specifically accurate straight-line distance between a point and the center point of the local projection
- Lambert conformal conic ([LCC](https://en.wikipedia.org/wiki/Lambert_conformal_conic_projection)) projections for regions covering thousands of kilometers, with the cone set to keep distance and area properties reasonable between the secant lines
- Stereographic ([STERE](https://en.wikipedia.org/wiki/Stereographic_projection)) projections for polar regions, but taking care not to rely on area and distance calculations thousands of kilometers from the center

<!--toDo:jn-->
<!--consider rewriting/updating the following section, maybe with some R code?-->
One possible approach to automatically select a projected CRS specific to a local dataset is to create an azimuthal equidistant ([AEQD](https://en.wikipedia.org/wiki/Azimuthal_equidistant_projection)) projection for the center-point of the study area.
This involves creating a custom CRS (with no EPSG code) with units of meters based on the center point of a dataset.
This approach should be used with caution: no other datasets will be compatible with the custom CRS created and results may not be accurate when used on extensive datasets covering hundreds of kilometers.

<!--toDo:jn-->
<!--consider rewriting/updating UTM section-->
A commonly used default is Universal Transverse Mercator ([UTM](https://en.wikipedia.org/wiki/Universal_Transverse_Mercator_coordinate_system)), a set of CRSs that divides the Earth into 60 longitudinal wedges and 20 latitudinal segments.
The transverse Mercator projection used by UTM CRSs is conformal but distorts areas and distances with increasing severity with distance from the center of the UTM zone.
Documentation from the GIS software Manifold therefore suggests restricting the longitudinal extent of projects using UTM zones to 6 degrees from the central meridian (source: [manifold.net](http://www.manifold.net/doc/mfd9/universal_transverse_mercator_projection.htm)).

Almost every place on Earth has a UTM code, such as "60H" which refers to northern New Zealand where R was invented.
UTM EPSG codes run sequentially from 32601 to 32660 for northern hemisphere locations and from 32701 to 32760 for southern hemisphere locations.



To show how the system works, let's create a function, `lonlat2UTM()` to calculate the EPSG code associated with any point on the planet as [follows](https://stackoverflow.com/a/9188972/): 


```r
lonlat2UTM = function(lonlat) {
  utm = (floor((lonlat[1] + 180) / 6) %% 60) + 1
  if(lonlat[2] > 0) {
    utm + 32600
  } else{
    utm + 32700
  }
}
```

The following command uses this function to identify the UTM zone and associated EPSG code for Auckland and London:




```r
epsg_utm_auk = lonlat2UTM(c(174.7, -36.9))
epsg_utm_lnd = lonlat2UTM(st_coordinates(london))
st_crs(epsg_utm_auk)$proj4string
#> [1] "+proj=utm +zone=60 +south +datum=WGS84 +units=m +no_defs"
st_crs(epsg_utm_lnd)$proj4string
#> [1] "+proj=utm +zone=30 +datum=WGS84 +units=m +no_defs"
```

Maps of UTM zones such as that provided by [dmap.co.uk](http://www.dmap.co.uk/utmworld.htm) confirm that London is in UTM zone 30U.

The principles outlined in this section apply equally to vector and raster datasets.
Some features of CRS transformation however are unique to each geographic data model.
We will cover the particularities of vector data transformation in Section \@ref(reproj-vec-geom) and those of raster transformation in Section \@ref(reprojecting-raster-geometries).

## Reprojecting vector geometries {#reproj-vec-geom}

\index{CRS!reprojection} 
\index{vector!reprojection} 
Chapter \@ref(spatial-class) demonstrated how vector geometries are made-up of points, and how points form the basis of more complex objects such as lines and polygons.
Reprojecting vectors thus consists of transforming the coordinates of these points.
This is illustrated by `cycle_hire_osm`, an `sf` object from **spData** that represents cycle hire locations across London.
The previous section showed how the CRS of vector data can be queried with `st_crs()`.
<!--toDo:rl-->
<!--not longer valid-->
<!-- Although the output of this function is printed as a single entity, the result is in fact a named list of class `crs`, with names `proj4string` (which contains full details of the CRS) and `epsg` for its code. -->
<!-- This is demonstrated below: -->



<!--toDo:rl-->
<!--not longer valid-->
<!-- This duality of CRS objects means that they can be set either using an EPSG code or a `proj4string`. -->
<!-- This means that `st_crs("+proj=longlat +datum=WGS84 +no_defs")` is equivalent to `st_crs(4326)`, although not all `proj4string`s have an associated EPSG code. -->
<!-- Both elements of the CRS are changed by transforming the object to a projected CRS: -->



<!--toDo:rl-->
<!--not longer valid-->
<!-- The resulting object has a new CRS with an EPSG code 27700. -->
<!-- But how to find out more details about this EPSG code, or any code? -->
<!-- One option is to search for it online. -->
<!-- Another option is to use a function from the **rgdal** package to find the name of the CRS: -->



<!--toDo:rl-->
<!--not longer valid-->
<!-- The result shows that the EPSG code 27700 represents the British National Grid, a result that could have been found by searching online for "[EPSG 27700](https://www.google.com/search?q=CRS+27700)". -->
<!-- But what about the `proj4string` element? -->
<!-- `proj4string`s are text strings that describe the CRS. -->
<!-- They can be seen as formulas for converting a projected point into a point on the surface of the Earth and can be accessed from `crs` objects as follows (see [proj.org/](https://proj.org/) for further details of what the output means): -->



\BeginKnitrBlock{rmdnote}<div class="rmdnote">Printing a spatial object in the console automatically returns its coordinate reference system.
To access and modify it explicitly, use the `st_crs` function, for example, `st_crs(cycle_hire_osm)`.</div>\EndKnitrBlock{rmdnote}


## Reprojecting raster geometries

\index{raster!reprojection} 
\index{raster!warping} 
\index{raster!transformation} 
\index{raster!resampling} 
The projection concepts described in the previous section apply equally to rasters.
However, there are important differences in reprojection of vectors and rasters:
transforming a vector object involves changing the coordinates of every vertex but this does not apply to raster data.
Rasters are composed of rectangular cells of the same size (expressed by map units, such as degrees or meters), so it is usually impracticable to transform coordinates of pixels separately.
Raster reprojection involves creating a new raster object, often with a different number of columns and rows than the original.
The attributes must subsequently be re-estimated, allowing the new pixels to be 'filled' with appropriate values.
In other words, raster reprojection can be thought of as two separate spatial operations: a vector reprojection of the raster extent to another CRS (Section \@ref(reproj-vec-geom)), and computation of new pixel values through resampling (Section \@ref(resampling)).
Thus in most cases when both raster and vector data are used, it is better to avoid reprojecting rasters and reproject vectors instead.

\BeginKnitrBlock{rmdnote}<div class="rmdnote">Reprojection of the regular rasters is also known as warping. 
Additionally, there is a second similar operation called "transformation".
Instead of resampling all of the values, it leaves all values intact but recomputes new coordinates for every raster cell, changing the grid geometry.
For example, it could convert the input raster (a regular grid) into a curvilinear grid.
The transformation operation can be performed in R using [the **stars** package](https://r-spatial.github.io/stars/articles/stars5.html).</div>\EndKnitrBlock{rmdnote}



The raster reprojection process is done with `project()` from the **terra** package.
Like the `st_transform()` function demonstrated in the previous section, `project()` takes a geographic object (a raster dataset in this case) and some CRS representation as the second argument.
On a side note -- the second argument can also be an existing raster object with a different CRS.

Let's take a look at two examples of raster transformation: using categorical and continuous data.
Land cover data are usually represented by categorical maps.
The `nlcd.tif` file provides information for a small area in Utah, USA obtained from [National Land Cover Database 2011](https://www.mrlc.gov/data/nlcd-2011-land-cover-conus) in the NAD83 / UTM zone 12N CRS.


```r
cat_raster = rast(system.file("raster/nlcd.tif", package = "spDataLarge"))
crs(cat_raster)
#> [1] "PROJCRS[\"NAD83 / UTM zone 12N\",\n    BASEGEOGCRS[\"NAD83\",\n        DATUM[\"North American Datum 1983\",\n            ELLIPSOID[\"GRS 1980\",6378137,298.257222101,\n                LENGTHUNIT[\"metre\",1]]],\n        PRIMEM[\"Greenwich\",0,\n            ANGLEUNIT[\"degree\",0.0174532925199433]],\n        ID[\"EPSG\",4269]],\n    CONVERSION[\"UTM zone 12N\",\n        METHOD[\"Transverse Mercator\",\n            ID[\"EPSG\",9807]],\n        PARAMETER[\"Latitude of natural origin\",0,\n            ANGLEUNIT[\"degree\",0.0174532925199433],\n            ID[\"EPSG\",8801]],\n        PARAMETER[\"Longitude of natural origin\",-111,\n            ANGLEUNIT[\"degree\",0.0174532925199433],\n            ID[\"EPSG\",8802]],\n        PARAMETER[\"Scale factor at natural origin\",0.9996,\n            SCALEUNIT[\"unity\",1],\n            ID[\"EPSG\",8805]],\n        PARAMETER[\"False easting\",500000,\n            LENGTHUNIT[\"metre\",1],\n            ID[\"EPSG\",8806]],\n        PARAMETER[\"False northing\",0,\n            LENGTHUNIT[\"metre\",1],\n            ID[\"EPSG\",8807]]],\n    CS[Cartesian,2],\n        AXIS[\"(E)\",east,\n            ORDER[1],\n            LENGTHUNIT[\"metre\",1]],\n        AXIS[\"(N)\",north,\n            ORDER[2],\n            LENGTHUNIT[\"metre\",1]],\n    USAGE[\n        SCOPE[\"unknown\"],\n        AREA[\"North America - 114°W to 108°W and NAD83 by country\"],\n        BBOX[31.33,-114,84,-108]],\n    ID[\"EPSG\",26912]]"
```

In this region, 8 land cover classes were distinguished (a full list of NLCD2011 land cover classes can be found at [mrlc.gov](https://www.mrlc.gov/data/legends/national-land-cover-database-2011-nlcd2011-legend)):


```r
unique(cat_raster)
#>       levels
#> 1      Water
#> 2  Developed
#> 3     Barren
#> 4     Forest
#> 5  Shrubland
#> 6 Herbaceous
#> 7 Cultivated
#> 8   Wetlands
```

When reprojecting categorical rasters, the estimated values must be the same as those of the original.
This could be done using the nearest neighbor method (`near`), which sets each new cell value to the value of the nearest cell (center) of the input raster.
An example is reprojecting `cat_raster` to WGS84, a geographic CRS well suited for web mapping.
The first step is to obtain the PROJ definition of this CRS, which can be done, for example using the [http://spatialreference.org](http://spatialreference.org/ref/epsg/wgs-84/) webpage. 
The final step is to reproject the raster with the `project()` function which, in the case of categorical data, uses the nearest neighbor method (`near`):


```r
cat_raster_wgs84 = project(cat_raster, "EPSG:4326", method = "near")
```

Many properties of the new object differ from the previous one, including the number of columns and rows (and therefore number of cells), resolution (transformed from meters into degrees), and extent, as illustrated in Table \@ref(tab:catraster) (note that the number of categories increases from 8 to 9 because of the addition of `NA` values, not because a new category has been created --- the land cover classes are preserved).


Table: (\#tab:catraster)Key attributes in the original ('cat\_raster') and projected ('cat\_raster\_wgs84') categorical raster datasets.

|CRS   | nrow| ncol|   ncell| resolution| unique_categories|
|:-----|----:|----:|-------:|----------:|-----------------:|
|NAD83 | 1359| 1073| 1458207|    31.5275|                 8|
|WGS84 | 1246| 1244| 1550024|     0.0003|                 9|

Reprojecting numeric rasters (with `numeric` or in this case `integer` values) follows an almost identical procedure.
This is demonstrated below with `srtm.tif` in **spDataLarge** from [the Shuttle Radar Topography Mission (SRTM)](https://www2.jpl.nasa.gov/srtm/), which represents height in meters above sea level (elevation) with the WGS84 CRS:


```r
con_raster = rast(system.file("raster/srtm.tif", package = "spDataLarge"))
crs(con_raster)
#> [1] "GEOGCRS[\"WGS 84\",\n    DATUM[\"World Geodetic System 1984\",\n        ELLIPSOID[\"WGS 84\",6378137,298.257223563,\n            LENGTHUNIT[\"metre\",1]]],\n    PRIMEM[\"Greenwich\",0,\n        ANGLEUNIT[\"degree\",0.0174532925199433]],\n    CS[ellipsoidal,2],\n        AXIS[\"geodetic latitude (Lat)\",north,\n            ORDER[1],\n            ANGLEUNIT[\"degree\",0.0174532925199433]],\n        AXIS[\"geodetic longitude (Lon)\",east,\n            ORDER[2],\n            ANGLEUNIT[\"degree\",0.0174532925199433]],\n    ID[\"EPSG\",4326]]"
```

We will reproject this dataset into a projected CRS, but *not* with the nearest neighbor method which is appropriate for categorical data.
Instead, we will use the bilinear method which computes the output cell value based on the four nearest cells in the original raster.^[Other methods mentioned in Section \@ref(resampling) also can be used here.]
The values in the projected dataset are the distance-weighted average of the values from these four cells:
the closer the input cell is to the center of the output cell, the greater its weight.
The following commands create a text string representing WGS 84 / UTM zone 12N, and reproject the raster into this CRS, using the `bilinear` method:


```r
con_raster_ea = project(con_raster, "EPSG:32612", method = "bilinear")
crs(con_raster_ea)
#> [1] "PROJCRS[\"WGS 84 / UTM zone 12N\",\n    BASEGEOGCRS[\"WGS 84\",\n        DATUM[\"World Geodetic System 1984\",\n            ELLIPSOID[\"WGS 84\",6378137,298.257223563,\n                LENGTHUNIT[\"metre\",1]]],\n        PRIMEM[\"Greenwich\",0,\n            ANGLEUNIT[\"degree\",0.0174532925199433]],\n        ID[\"EPSG\",4326]],\n    CONVERSION[\"UTM zone 12N\",\n        METHOD[\"Transverse Mercator\",\n            ID[\"EPSG\",9807]],\n        PARAMETER[\"Latitude of natural origin\",0,\n            ANGLEUNIT[\"degree\",0.0174532925199433],\n            ID[\"EPSG\",8801]],\n        PARAMETER[\"Longitude of natural origin\",-111,\n            ANGLEUNIT[\"degree\",0.0174532925199433],\n            ID[\"EPSG\",8802]],\n        PARAMETER[\"Scale factor at natural origin\",0.9996,\n            SCALEUNIT[\"unity\",1],\n            ID[\"EPSG\",8805]],\n        PARAMETER[\"False easting\",500000,\n            LENGTHUNIT[\"metre\",1],\n            ID[\"EPSG\",8806]],\n        PARAMETER[\"False northing\",0,\n            LENGTHUNIT[\"metre\",1],\n            ID[\"EPSG\",8807]]],\n    CS[Cartesian,2],\n        AXIS[\"(E)\",east,\n            ORDER[1],\n            LENGTHUNIT[\"metre\",1]],\n        AXIS[\"(N)\",north,\n            ORDER[2],\n            LENGTHUNIT[\"metre\",1]],\n    USAGE[\n        SCOPE[\"unknown\"],\n        AREA[\"World - N hemisphere - 114°W to 108°W - by country\"],\n        BBOX[0,-114,84,-108]],\n    ID[\"EPSG\",32612]]"
```

Raster reprojection on numeric variables also leads to small changes to values and spatial properties, such as the number of cells, resolution, and extent.
These changes are demonstrated in Table \@ref(tab:rastercrs)^[
Another minor change, that is not represented in Table \@ref(tab:rastercrs), is that the class of the values in the new projected raster dataset is `numeric`.
This is because the `bilinear` method works with continuous data and the results are rarely coerced into whole integer values.
This can have implications for file sizes when raster datasets are saved.
]:


Table: (\#tab:rastercrs)Key attributes in the original ('con\_raster') and projected ('con\_raster\_ea') continuous raster datasets.

|CRS          | nrow| ncol|  ncell| resolution| mean|
|:------------|----:|----:|------:|----------:|----:|
|WGS84        |  457|  465| 212505|     0.0008| 1843|
|UTM zone 12N |  515|  422| 217330|    83.5334| 1842|

\BeginKnitrBlock{rmdnote}<div class="rmdnote">Of course, the limitations of 2D Earth projections apply as much to vector as to raster data.
At best we can comply with two out of three spatial properties (distance, area, direction).
Therefore, the task at hand determines which projection to choose. 
For instance, if we are interested in a density (points per grid cell or inhabitants per grid cell) we should use an equal-area projection (see also Chapter \@ref(location)).</div>\EndKnitrBlock{rmdnote}


## Modifying map projections

<!--toDo:jn-->
<!--not longer valid-->
<!-- proj4strings still can be used - explain how to use them and when -->
<!-- however, focus on wkt2 customization here! -->
<!-- also, consider moving this section to the bottom of the chapter and show some raster examples -->

<!-- \index{CRS!proj4string}  -->
<!-- Established CRSs captured by EPSG codes are well-suited for many applications. -->
<!-- However in some cases it is desirable to create a new CRS, using a custom `proj4string`. -->
<!-- This system allows a very wide range of projections to be created, as we'll see in some of the custom map projections in this section. -->

<!-- A long and growing list of projections has been developed and many of these can be set with the `+proj=` element of `proj4string`s.^[ -->
<!-- The Wikipedia page 'List of map projections' has 70+ projections and illustrations. -->
<!-- ] -->

<!-- When mapping the world while preserving area relationships, the Mollweide projection is a good choice [@jenny_guide_2017] (Figure \@ref(fig:mollproj)). -->
<!-- To use this projection, we need to specify it using the `proj4string` element, `"+proj=moll"`, in the `st_transform` function: -->





On the other hand, when mapping the world, it is often desirable to have as little distortion as possible for all spatial properties (area, direction, distance).
One of the most popular projections to achieve this is the Winkel tripel projection (Figure \@ref(fig:wintriproj)).^[
This projection is used, among others, by the National Geographic Society.
]
`st_transform_proj()` from the **lwgeom** package allows for coordinate transformations to a wide range of CRSs, including the Winkel tripel projection:


```r
world_wintri = lwgeom::st_transform_proj(world, crs = "+proj=wintri")
```

<div class="figure" style="text-align: center">
<img src="07-reproj_files/figure-html/wintriproj-1.png" alt="Winkel tripel projection of the world." width="100%" />
<p class="caption">(\#fig:wintriproj)Winkel tripel projection of the world.</p>
</div>





<!-- Moreover, PROJ parameters can be modified in most CRS definitions. -->
<!-- The below code transforms the coordinates to the Lambert azimuthal equal-area projection centered on longitude and latitude of `0` (Figure \@ref(fig:laeaproj1)). -->


<!-- plot(world_laea1$geom) -->
<!-- plot(world_laea1$geom, graticule = TRUE) -->



<!-- We can change the PROJ parameters, for example the center of the projection, using the `+lon_0` and `+lat_0` parameters.  -->
<!-- The code below gives the map centered on New York City (Figure \@ref(fig:laeaproj2)). -->





More information on CRS modifications can be found in the [Using PROJ](https://proj.org/usage/index.html) documentation.

There is more to learn about CRSs.
An excellent resource in this area, also implemented in R, is the website R Spatial.
Chapter 6 from this free online book is recommended reading --- see: [rspatial.org/terra/spatial/6-crs.html](https://rspatial.org/terra/spatial/6-crs.html)

## Exercises


E1. Create a new object called `nz_wgs` by transforming `nz` object into the WGS84 CRS.

- Create an object of class `crs` for both and use this to query their CRSs.
- With reference to the bounding box of each object, what units does each CRS use?
- Remove the CRS from `nz_wgs` and plot the result: what is wrong with this map of New Zealand and why?



E2. Transform the `world` dataset to the transverse Mercator projection (`"+proj=tmerc"`) and plot the result.
What has changed and why?
Try to transform it back into WGS 84 and plot the new object.
Why does the new object differ from the original one?



E3. Transform the continuous raster (`con_raster`) into NAD83 / UTM zone 12N using the nearest neighbor interpolation method.
What has changed?
How does it influence the results?



E4. Transform the categorical raster (`cat_raster`) into WGS 84 using the bilinear interpolation method.
What has changed?
How does it influence the results?



<!--toDo:jn-->
<!--improve/replace/modify the following q-->
<!-- E5. Create your own `proj4string`.  -->
<!-- It should have the Lambert Azimuthal Equal Area (`laea`) projection, the WGS84 ellipsoid, the longitude of projection center of 95 degrees west, the latitude of projection center of 60 degrees north, and its units should be in meters. -->
<!-- Next, subset Canada from the `world` object and transform it into the new projection.  -->
<!-- Plot and compare a map before and after the transformation. -->

<!-- ```{r 06-reproj-40} -->
<!-- new_p4s = "+proj=laea +ellps=WGS84 +lon_0=-95 +lat_0=60 +units=m" -->
<!-- canada = dplyr::filter(world, name_long == "Canada") -->
<!-- new_canada = st_transform(canada, new_p4s) -->
<!-- par(mfrow = c(1, 2)) -->
<!-- plot(st_geometry(canada), graticule = TRUE, axes = TRUE) -->
<!-- plot(st_geometry(new_canada), graticule = TRUE, axes = TRUE) -->
<!-- ``` -->