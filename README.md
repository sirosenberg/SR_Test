# Guide to creating data tilesets
#### (aka, the Tippecanoe cheat sheet)

This process may evolve as we hear back from mapbox on a handful of issues.
There are three main steps to creating the broadband-data tilesets:
1. Join the tabular data to the geometry
1. Create tilesets for different data sets and zoom levels with different settings
1. Combine tilesets where possible to make tileset management easier.

For broadband data and tilesets, the data sources and tippecanoe settings will vary based on zoom level.  

Zoom level | geometry data source | tabular data source | tippecanoe settings
-----------|----------------------|---------------------|---------------------
0-2|cartographic (500k) counties | county_numprov rounded to .x | county set 1 (see below)
3-4|cartographic (500k) counties | county_numprov rounded to .x | county set 2
5-8|cartographic (500k) tracts | tract_numprov rounded to .x | tract set 1, one for each speed
9a | tracts, excluding water blocks | tract_numprov rounded to .x | tract set 2
9b | blocks in tracts >2.e9 m<sup>2</sup>| block_numprov | block set 2
10 | blocks | block_numprov | block set 1
11-14 | blocks | block_numprov | block set 2

## Code used to create tilesets (as of 17Nov17)
#### The remainder of the document tries to explain the process leading up to these commands and the choices made


## Using `join-data` to join geometry and tabular data
We need to join the tabular data to the geometry *before* running `tippecanoe` because of the way `tippecanoe` is designed.  The limit on each vector tile is 500 kB, inclusive of data.  If you create the tileset first, then add substantial amounts of data to the tiles, as would be the case using the `tippecanoe` then `tile-join` process described on the mapbox github page, you would have many tiles dropped for exceeding the 500k limit (since `tile-join` lacks the code to simplify that `tippecanoe` has).

`join-data` is a "quick hack" perl script written by mapbox (Eric Fischer) to allow us to join the data first, then run tippecanoe.  Mapbox is looking into a longer-term solution, but for the time being, there are a number of issues limitations when using `join-data`.  

`join-data` performs an inner join between the geometry and the tabular data.  So if we want to pull only some geometry data into a tileset (e.g., block shapes only for large tracts in zoom level 9), we do not need to create a geojson file with a subset of the geometry data, we only need to create a csv with the records of interest.

The output of `join-data` is to standard output, which then gets piped into tippecanoe `| tippecanoe...` or into a gzip file (see issue 4 below).

**Issue 1:** *Join-data requires both the geojson and the csv data to be sorted by FIPS code in ascending order.*

To ensure the data are sorted properly, there are a couple of options:

* Sort the data when first writing/creating the file (e.g., include `.sort_values` in `df.to_csv`)
* Use the `sort` command
  * The unix `sort` command sorts on the first column unless otherwise specified; you can specify a later column using the `-k` option, but note that sort looks for space-delimited text (i.e., you need to find what *space-delimited* column the FIPS code occupies in the geojson file).
  * You will need to sort excluding the header (i.e., to keep the header as the first line of the file). You can do this by adding a space to the first line using `sed`, then later removing that space with `sed` (or remembering that you first column name starts with a space when reading into pandas); or by using `head` to write the header first, then use `tail` to grab the remaining lines for the sort
  * Taken together the command would look like:
    * `sed '1s/^/ /' block_numprov.csv | sort |sed '1s/^.//'> block_numprov_sort.csv`
or
    * `head -n 1 block_numprov.csv> block_numprov _sort.csv; tail -n +2 block_numprov.csv | sort >> block_numprov_sort.csv`

  * It appears the sort command can shuffle rows, so, e.g., the projection information moves to the end of the file. This does not appear to be a problem for Tippecanoe, but would probably render the resulting geojson unsuitable for anything else.

**Issue 2:** *Join-data looks for the field name(s) of the spatial-identifier/FIPS code of the geojson* 

This happens in the `join-data` script on line 19 (default is “geoid10” and “tract_id2010”). Edit the join-data script to set one of these to be the same as the FIPS identifier in the geojson data if different (or match the header name to `join-data` when creating the csv in pandas).

`Join-data` appears to match the first column of the csv data. If that column has the same name as the FIPS code in the geojson file, that identifier is dropped (which is fine). If it is different, you need to (apparently) manually remove that field using the -x option of Tippecanoe to avoid passing an unnecessary, and large, data field.

**Issue 3:** *It’s important to minimize the data payload for Tippecanoe.* 

Tippecanoe stores data by creating a table of all values across all geometries in the tile, and uses (single byte) pointers from the geometries to that table (at least conceptually; see https://github.com/mapbox/vector-tile-spec/tree/master/2.1#44-feature-attributes). Thus having fewer numbers that repeat more often means saving on the data payload. 
* For tract and county data, which are floating point, this makes rounding very important.  Rounding to 1 digit after the decimal means a smaller data payload and therefore fewer/smaller changes to the geometry data by `tippecanoe`.
* For counties, which have integer data, this is a smaller problem (though there may be some benefit to limiting the range of integer values).  However, it appears that `data-join` drops NULL values (both the integer and the field name), which provides significant savings.  Thus we want to have the block_numprov table include NULLs instead of 0 values; and then use the "default" value in styling (rather than looking for a 0 value).  

**Issue 4:** *`join-data` can take a long time to run (a few hours) for block data*
To avoid running the join twice, you can save the output of `join-data` to a file, though the time to run it for county and tract data is quite short.  Since the output of `join-data` is simply a geojson with the data fields added, it would be an extremely large geojson for block data (~70 GB).  However, you can save the ouput to a gzip file, then stream the output of gunzip to tippecanoe:
 * Saving the file: `data-join file1.geojson file1.csv | gzip -9 > joined.geojson.gz`
 * Using the saved file: `gzip -dc joined.geojson.gz | tippecanoe …` (note that `gzip -d` is the same as `gunzip`)

## `tippecanoe` settings to create tilesets
There are a number of options in `tippecanoe` designed to ensure that each tile size remains below the 500k limit. The difficulty is in finding a combination of those options that provides enough reduction in tile size without creating odd visual artifacts or gaps (e.g., all the --drop flags will open up gaps in the data and can’t be used for these data).

### Removing unneeded data
As noted above, the tilesize includes the data payload, so it's important to drop any unnecessary fields using the `-x` or `-X` flags on `tippecanoe`.  Exactly what fields need to be excluded using `-x` (or included using `-y`) depends on the original source for the geojson and what fields it includes beyond the FIPS code and geometry.  For broadband data, we only want to include the 315 broadband values that matter for county and tract data, and the 315 broadband values plus the h2only_undev field.  For the state, county, CBSA, CD, CDP and tribal areas, we need the ID associated with each geography's geometry; and for the provider map, we need the hoconum.  **Any other data fields should be dropped.**

### Reducing tile size by removing geometry data...without making it look bad.
By default, `tippecanoe` reduces the level of detail for individual tiles when a tile is larger than 500k to try and reduce the tile size. It will continue to reduce the level of detail until it fits, or it gets to a detail level of 7 (a tile of 128 pixels across) and fails. This reduction in detail can cause odd visual artifacts, especially at low levels of detail. The reduction in size comes from greater simplification (see description of simplification below).

Given the problems with the reduction in detail, and to avoid opening up gaps using the various `-drop` flags, this process relies on `--coalesce` and `--coalesce-smallest-as-needed` to reduce the size of tiles.

The `--coalesce` flag aggregates together geometries with the same field values (in our case, across all 315/316 variables) that are adjacent, basically performing the equivalent of ST_Union in PostGIS and dissolving the internal boundaries. This is among the reasons why it’s important to drop the FIPS code from the data set – since each geometry has a unique value, the FIPS field would otherwise prevent `--coalesce` from having any effect (the other reason is that it’s more data payload which requires even greater savings from reducing the geometry). Using `--coalesce` means there is less use of the `--coalesce-smallest-as-needed` flag. The downside to using `--coalesce` is that you lose the boundaries between blocks and can create relatively large blobs. Note that `--coalesce` is likely to have little effect on tract and county data since those are floating point numbers (i.e., they have more variation in value).

The `--coalesce-smallest-as-needed` flag simply identifies the smallest-area geometries and merges them into their neighbors. Right now, the combined geometry has the data values of the larger geometry. That can and does change the values for some areas and so can change the appearance. Thus the use of `--coalesce` to minimize the number of blocks that might change their value.

**Right now, `--coalesce-smallest-as-needed` is creating unexpected outputs in zoom 10 (where it is critical)**

#### How `tippecanoe` simplifies geometry data
The distance represented by each pixel depends on the zoom level and the number of pixels per tile.

Zoom level 0 incorporates the whole earth, and each zoom level higher represents a 2x gain in resolution in x and in y. Zoom level 1 is a 2x2 grid covering the earth; zoom level 2 is a 4x4 grid. So the degrees of lat/lon covered by each tile is ~360/(2^(zoom level).

The number of pixels in each tile is controlled by the level of detail – the number of pixels is 2^detail level. So, for the default detail 12, there are 2^12 pixels or 4096 pixels. Taken with the zoom level, the resolution of each pixel is ~360/(2^(zoom level + detail)) in degrees. Taking ~0.00000274 feet per degree (at the equator), and you get the distance per pixel, which depends on zoom level and detail level. So, e.g., at zoom level 10 and detail level 12, each pixel is ~(360/(2^(10+12))/.00000274) = 32 feet.

#### Simplification

By default, the resolution per pixel is the distance that Tippecanoe uses in applying the Douglas-Peucker simplification (this is the same simplification in ST_Simplify and the default algorithm used in ArcGIS Pro). The `-S` command overrides this distance and provides a value to multiply that distance by for the simplification; i.e., a value of `-S 8` would use a distance of 8x32 feet at zoom level 10 and detail level 12.

By default, the simplification modifies each polygon independently. This means that at higher simplification levels, gaps can open up between polygons, which is a problem for data like blocks, tracts and counties which should have complete coverage. The `--detect-shared-borders` flag is meant to prevent this from happening. [it appears there’s a bug in the implementation so that when detect-shared-borders is set and the level of detail decreases gaps will still open up. See https://github.com/mapbox/tippecanoe/issues/482].

### Overzoom to avoid creating higher-zoom data than needed

The `-d` flag sets the detail level at the highest zoom (the default is 12). By setting `-d 14`, you are adding 4x as much resolution to the highest zoom level (set with the `-z` option), which means that as you zoom in past the highest zoom level, you have the resolution as if you had created two additional zoom levels. The advantage to using the `-d` option is that the resulting mbtiles file is much smaller (and is created much faster) than creating two additional zoom levels.

## Using `tile-join` to combine tilesets
We can reduce the number of tilesets by combining two tilesets into one in some instances using `tile-join`.  The 500k-per-tile limit applies when using `tile-join`.  So where we have tilesets that are at different zoom levels -- i.e., where overlaying two different tilesets won't create a tile that exceeds 500k -- we can combine tilesets.  This works for county data and block data, which have tilesets at different zoom levels; it also seems to work for tract and block data at zoom 9.  However, it will not work for tract data at zoom levels 5-8 since the tileset for each speed can approach 500k.  The syntax to combine two tilesets into one is
`tile-join 
