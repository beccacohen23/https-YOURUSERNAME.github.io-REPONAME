---
layout: post
title: Bringing Your Maps into Focus
tags: [gdal, geospatial, gis, wms, webmapping]
date: 2016-06-13 13:00:00
updated: 2021-01-19 13:00:00
authors:
    - jayvarner
    - megan
---

Here at Emory, we have some pretty cool old [maps of Atlanta](http://www.digitalgallery.emory.edu/luna/servlet/view/all/where/Atlanta?sort=title%2Cpage_no_%2Ccity%2Cdate) that we wanted to share in a more interactive way. But when we tried, the maps looked like this:

<p data-height="500" data-theme-id="0" data-slug-hash="eExoyO" data-default-tab="result" data-user="jayvarner" data-embed-version="2" data-pen-title="eExoyO" class="codepen">See the Pen <a href="https://codepen.io/jayvarner/pen/eExoyO/">eExoyO</a> by Jay Varner (<a href="https://codepen.io/jayvarner">@jayvarner</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://production-assets.codepen.io/assets/embed/ei.js"></script>

After a good bit of research, lots of trial and error, and working with our colleagues [Eric Willoughby](http://library.gsu.edu/profile/eric-willoughby/) at Georgia State University and [Amanda Henley](https://twitter.com/gislibrarian) at the University of North Carolina, we came up with a process that gave us this:

<p data-height="500" data-theme-id="0" data-slug-hash="brzJLx" data-default-tab="result" data-user="jayvarner" data-embed-version="2" data-pen-title="brzJLx" class="codepen">See the Pen <a href="https://codepen.io/jayvarner/pen/brzJLx/">brzJLx</a> by Jay Varner (<a href="https://codepen.io/jayvarner">@jayvarner</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://production-assets.codepen.io/assets/embed/ei.js"></script>


## What We Did

We [georeferenced](https://en.wikipedia.org/wiki/Georeference) our map in the appropriate [coordinate system](https://en.wikipedia.org/wiki/Coordinate_system) for the map’s area. In our case [EPSG:2240](http://spatialreference.org/ref/epsg/2240/). We’ll refer to specific coordinate systems with their [EPSG code](https://en.wikipedia.org/wiki/International_Association_of_Oil_%26_Gas_Producers#European_Petroleum_Survey_Group).

Next we took the resulting [GeoTIFF](https://en.wikipedia.org/wiki/GeoTIFF) and uploaded it to our [GeoServer](http://geoserver.org) so the map could be shared via [WMS](https://en.wikipedia.org/wiki/Web_Map_Service).

Finally, like the examples above, we used [Leaflet](http://leafletjs.com/) to display the maps in the browser overlaid on a modern street map.

## What We Did *Wrong*
Our first problem was that the GeoTIFF is a high resolution image. In the first example on this page, we were squeezing a 5,931 x 5,779 pixel image (the full resolution) into a 326 x 276 pixel box (for the zoom level shown above). There’s just too much data to display clearly. Notice that if you zoom all the way in to our original example, the map looks great.

<p data-height="500" data-theme-id="0" data-slug-hash="OjdGvE" data-default-tab="result" data-user="jayvarner" data-embed-version="2" data-pen-title="OjdGvE" class="codepen">See the Pen <a href="https://codepen.io/jayvarner/pen/OjdGvE/">OjdGvE</a> by Jay Varner (<a href="https://codepen.io/jayvarner">@jayvarner</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://production-assets.codepen.io/assets/embed/ei.js"></script>

Our other problem was that we used a coordinate system that is inappropriate for web display. Tiled base maps that are provided by organizations like [OpenStreetMap](http://www.openstreetmap.org/about) are based on [EPSG:3857](http://spatialreference.org/ref/sr-org/7483/), aka [WGS 84  Web Mercator](https://en.wikipedia.org/wiki/Web_Mercator#EPSG:3857).

## GDAL Can Solve those Problems

"[GDAL](http://www.gdal.org/) is a translator library for raster and vector geospatial data formats..." This is our ~~three~~ two[^fewer] step process.

### gdalwarp

We use `gdalwarp` to reproject, resample, add internal tiling, and compress.

[^fewer]: In previous talks and workshops, we presented a three-step process where we did the tiling and compressed the image using `gdal_translate`. While writing this post we realized a way to combine these processes with `gdalwarp`.

#### Reproject

When you georeference a scanned map, you select a [projection](https://en.wikipedia.org/wiki/Map_projection). For most GIS needs, you will want to use the projection appropriate for the chunk of Earth your map covers. However, for displaying maps on the web, you want to use EPSG:3857[^whcihcode].

[^whcihcode]:There is confusion between EPSG:3857 and EPSG:4326. For displaying raster data using something like Leaflet, OpenLayers, etc. 3857 will result in a much clearer image. Here are two links that helped us understand. [http://gis.stackexchange.com/a/48952](http://gis.stackexchange.com/a/48952) and [http://www.faqoverflow.com/gis/48949.html](http://www.faqoverflow.com/gis/48949.html)

#### Resample

`gdalwarp` offers various algorithms[^man] for [resampling](https://en.wikipedia.org/wiki/Image_scaling). We suggest you try a few and see what works best. We’ve had good results using `average`, but many folks will suggest `near`.

[^man]: You can see a list of available algorithms by running `man gdalwarp` and reading the explanation of the `-r` option. `man` is short for manual. Use the space bar to scroll through the text and type `q` to quit the man page viewer. Or you can [read it here](http://www.gdal.org/gdalwarp.html).

#### Tiling

GeoTIFFs are organized in 1 pixel strips by default. OpenStreetMap tiles are 256x256. We match OpenStreetMap by adding internal tiles. We’ll go deeper into tiling in our follow up [post](/2016/06/13/an-overview-of-overviews).

#### Compress

GeoTIFFs can be huge and it's not very nice to make people load large files when they don't have to. GDAL can also compress our GeoTIFFs. There are many compression algorithms, but we’re pretty happy with the results we’re getting with `JPEG`.

#### The Command

Here is an example showing how to set all the options. Note the last argument is a path to a file the command will create. Your original GeoTIFF will be preserved.

##### GDALWARP Syntax

~~~shell
gdalwarp -s_srs <source ESPG> -t_srs <target EPSG> -r average -co 'TILED=YES/NO' -co 'BLOCKXSIZE=XXX' -co 'BLOCKYSIZE=XXX' -co 'COMPRESS=ABC' </path/to/source/geo.tif> </path/to/new/geo.tif>
~~~

##### GDALWARP Example

~~~shell
gdalwarp -s_srs EPSG:2240 -t_srs EPSG:3857 -r average -co 'TILED=YES' -co 'BLOCKXSIZE=256' -co 'BLOCKYSIZE=256' -co 'COMPRESS=JPEG' atlanta_1928_sheet45.tif processed/atlanta_1928_sheet45.tif
~~~

_NOTE_: We are creating a new copy of the GeoTIFF in a directory called "processed". You can save it where ever you like.

Now you have a new GeoTIFF projected in Web Mercator and preserved your original.

##### About Transparency and JPEG Compression

If the GeoTIFF is not a perfect rectangle, or if its orientation is not straight north/south or east/west, a black or white border will fill the empty space[^examples]. In short, the JPEGs do not have a fourth band for transparency. You *can* add the fourth band to a JPEG compressed GeoTIFF, but it negates the file size savings. To include the fourth alpha band replace the `-co 'COMPRESS=JPEG'` option in the above command with `-srcnodata 0 -dstalpha`[^alpha].

[^examples]: The map [in this example](https://codepen.io/jayvarner/full/mdBYRyR) not perfectly straight and compressed with JPEG. [This is ths same map](https://codepen.io/jayvarner/full/PoJvWPK) not using JPEG compression and has the alpha band.

[^alpha]: gdalwarp -s_srs EPSG:2240 -t_srs EPSG:3857 -r average -srcnodata 0 -dstalpha -co 'TILED=YES' -co 'BLOCKXSIZE=256' -co 'BLOCKYSIZE=256' atlanta_1928_sheet45.tif processed/atlanta_1928_sheet45.tif

### Add Overviews

We discuss overviews in greater depth in our follow-up [post](/2016/06/13/an-overview-of-overviews). Remember when we said the main reason the map looked so bad was because there was just too much data? Well, overviews fix that. What happens here is we create internal versions of our image that are smaller and lower resolution.

We also have to tell it to keep our blocks, or internal tiles, at 256, the default is 128. You can set `GDAL_TIFF_OVR_BLOCKSIZE` as an [environment variable](https://help.ubuntu.com/community/EnvironmentVariables), or just override the default with the `--config` option.

#### GDALADDO Syntax

~~~shell
gdaladdo --config GDAL_TIFF_OVR_BLOCKSIZE XXX -r average </path/to/new.tif> levels (list of numbers)
~~~

#### GDALADDO Example

```shell
gdaladdo --config GDAL_TIFF_OVR_BLOCKSIZE 256 -r average processed/atlanta_1928_sheet45.tif 2 4 8 16 32
```

## Automation
Obviously if you have more than two maps to process, running these commands on each will get real old real fast. Fortunately, since GDAL is a command line tool, we can automate the whole process. There are wrappers for GDAL in many languages: [Python](https://pcjericks.github.io/py-gdalogr-cookbook/), [PHP](https://github.com/geonef/php5-gdal), [.Net](https://gdalnet.codeplex.com/), [Ruby](https://github.com/zhm/gdal-ruby) (though the Ruby gem is no longer maintained and not updated for GDAL 2), etc.

We have [automated our process](https://github.com/zhm/gdal-ruby). It’s nice to just run a script and go enjoy some coffee while the computer does all the work.

## Conclusion
This has been an evolving process. We found ways to improve the process every time we prepared for a workshop, conference talk, or writing this post. We would love to hear any feedback or suggestions.
