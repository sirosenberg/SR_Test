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
9b | blocks in tracts >2.e9 m<sup>2</sup>| block_numprov | big-block set
10 | blocks | block_numprov | block set 1
11-14 | blocks | block_numprov | block set 2

## Code used to create tilesets (as of 17Nov17)
#### The remainder of the document tries to explain the process leading up to these commands and the choices made

Note that at the end of the `tippecanoe` command, there's `2>&1 | tee filename.log` which redirects the output of `tippecanoe` from the standard error to standard out (`2>&1`), then splits the output to standard out and to a file (`| tee filename`) to preserve the run-time messages from `tippecanoe` about what tiles required additional reductions.

### County
```
# county set 1
./join-data-county.pl geojsons/us_counties_2010_500k_sort.geojson csvs/county_data_sort_round.csv | time tippecanoe -P -Z 0 -z 2 -S 8 --detect-shared-borders --coalesce-smallest-as-needed -x geoid10 -l county_round_z0_z2 -f -o mbtiles/county_round_z0_z2.mbtiles 2>&1 | tee ./county_z02.log
# county set 2
./join-data-county.pl geojsons/us_counties_2010_500k_sort.geojson csvs/county_data_sort_round.csv | time tippecanoe -P -Z 3 -z 4 --detect-shared-borders --coalesce-smallest-as-needed -x geoid10 -l county_round_z3_z4 -f -o mbtiles/county_round_z3_z4.mbtiles 2>&1 | tee ./county_z34.log
tile-join -f -o mbtiles/county_round.mbtiles mbtiles/county_round_z0_z2.mbtiles mbtiles/county_round_z3_z6.mbtiles
```
### Tracts
This code creates the tract tiles through zoom 9, though with the block-tile layer, we only need to create through zoom 8.  It is possible to combine the tract tiles for all speeds at zoom 7, but not at zooms 5 or 6.
```
# tract set 1 (five different speeds)
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_200.csv| time tippecanoe -P -Z 4 -z 5 -S 8 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_200_z4_z5 -f -o mbtiles/tract_round_200_z4_z5.mbtiles 2>&1 | tee ./tract200_z4.log
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_10_1.csv| time tippecanoe -P -Z 4 -z 5 -S 8 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_10_1_z4_z5 -f -o mbtiles/tract_round_10_1_z4_z5.mbtiles 2>&1 | tee ./tract10_z4.log
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_25_3.csv| time tippecanoe -P -Z 4 -z 5 -S 8 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_25_3_z4_z5 -f -o mbtiles/tract_round_25_3_z4_z5.mbtiles 2>&1 | tee ./tract25_z4.log
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_50_5.csv| time tippecanoe -P -Z 4 -z 5 -S 8 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_50_5_z4_z5 -f -o mbtiles/tract_round_50_5_z4_z5.mbtiles 2>&1 | tee ./tract50_z4.log
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_100_10.csv| time tippecanoe -P -Z 4 -z 5 -S 8 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_100_10_z4_z5 -f -o mbtiles/tract_round_100_10_z4_z5.mbtiles 2>&1 | tee ./tract100_z4.log

# tract set 2 (five different speeds)
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_200.csv| time tippecanoe -P -Z 6 -z 9 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_200_z6_z9 -f -o mbtiles/tract_round_200_z6_z9.mbtiles 2>&1 | tee ./tract200_z6.log
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_10_1.csv| time tippecanoe -P -Z 6 -z 9 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_10_1_z6_z9 -f -o mbtiles/tract_round_10_1_z6_z9.mbtiles 2>&1 | tee ./tract10_z6.log
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_25_3.csv| time tippecanoe -P -Z 6 -z 9 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_25_3_z6_z9 -f -o mbtiles/tract_round_25_3_z6_z9.mbtiles 2>&1 | tee ./tract25_z6.log
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_50_5.csv| time tippecanoe -P -Z 6 -z 9 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_50_5_z6_z9 -f -o mbtiles/tract_round_50_5_z6_z9.mbtiles 2>&1 | tee ./tract50_z6.log
./join-data-tract.pl geojsons/us_tracts_2010_500k_4326_sort.geojson csvs/tract_data_sort_round_100_10.csv| time tippecanoe -P -Z 6 -z 9 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x STATE -x COUNTY -x TRACT -x NAME -x LSAD -x Tract -x CENSUSAREA -x tract_id2010 -l tract_round_100_10_z6_z9 -f -o mbtiles/tract_round_100_10_z6_z9.mbtiles 2>&1 | tee ./tract100_z6.log

tile-join -f -o mbtiles/tract_round_200.mbtiles mbtiles/tract_round_200_z4_z5.mbtiles mbtiles/tract_round_200_z6_z9.mbtiles
tile-join -f -o mbtiles/tract_round_10_1.mbtiles mbtiles/tract_round_10_1_z4_z5.mbtiles mbtiles/tract_round_10_1_z6_z9.mbtiles
tile-join -f -o mbtiles/tract_round_25_3.mbtiles mbtiles/tract_round_25_3_z4_z5.mbtiles mbtiles/tract_round_25_3_z6_z9.mbtiles
tile-join -f -o mbtiles/tract_round_50_5.mbtiles mbtiles/tract_round_50_5_z4_z5.mbtiles mbtiles/tract_round_50_5_z6_z9.mbtiles
tile-join -f -o mbtiles/tract_round_100_10.mbtiles mbtiles/tract_round_100_10_z4_z5.mbtiles mbtiles/tract_round_100_10_z6_z9.mbtiles
```
### Tract-block for zoom 9
```
# tract set 2 (using non-cartographic tracts)
./join-data-tract2.pl geojsons/us_tracts_2010_4326.geojson csvs/tract_data_sort_round_all.csv| time tippecanoe -P -Z 9 -z 9 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x tract_id2010 -l tract_round_all_z9 -f -o mbtiles/tract_round_all_z9.mbtiles 2>&1 | tee ./tract200_z9.log
# big-block set
./join-data.pl geojsons/big_tract_blocks.geojson csvs/block_numprov_full_null.csv| time tippecanoe -P -Z 9 -z 9 --detect-shared-borders --coalesce --coalesce-smallest-as-needed  -x block_fips -l big_tract_blocks_z9 -f -o mbtiles/big_tract_blocks_z9.mbtiles 2>&1 | tee ./big_tract_block_z9.log

tile-join -f -o mbtiles/all_zoom9.mbtiles mbtiles/tract_round_all_z9.mbtiles mbtiles/big_tract_blocks_z9.mbtiles
```
The following code was used to create the geojson tract data for the tract-block layer at zoom 9 as a short-term measure.  ***This should be replaced with geopandas (in the case of the non-cartographic tract geojson).***
`ogr2ogr -f GeoJSON ./us_tracts_2010_4326.geojson PG:"host=gisp-proc-wcb-pg-int.cfddrd5nduv6.us-west-2.rds.amazonaws.com user=wcb_srosenberg dbname=wcb_internal password=wcb_srosenberg" -sql "SELECT tract_fips, geom FROM census2010.tract_land_4326"`

The following code was used to create a geojson file for blocks in large tracts as a short-term measure.  Since we can use the general block geojson data, and simply create a csv with data only for blocks in large tracts, ***this should be replaced with pandas to create that kind of csv.***
`ogr2ogr -f GeoJSON ./big_tract_blocks.geojson PG:"host=gisp-proc-wcb-pg-int.cfddrd5nduv6.us-west-2.rds.amazonaws.com user=wcb_srosenberg dbname=wcb_internal password=wcb_srosenberg" -sql "SELECT geoid10, geom FROM census2010.block_us WHERE aland10>0 AND left(geoid10,11) IN (SELECT tract_fips FROM census2010.tract_land_4326 WHERE land_area_m > 2e9) ORDER BY geoid10"` 

### Blocks
Note that this code does not save the output of join-data into a gzipped geojson, which might make the second run complete more quickly.  See Issue #4 of `join-data` section below.
```
# block set 1
./join-data.pl geojsons/block_us.geojson csvs/block_numprov_full_null.csv | time tippecanoe -P -Z 10 -z 10 -S 8 --detect-shared-borders -x block_fips --coalesce --coalesce-smallest-as-needed -l block_null_z10 -f -o mbtiles/block_null_z10.mbtiles 2>&1 | tee ./block_null_z10.log
# block set 2
./join-data.pl geojsons/block_us.geojson csvs/block_numprov_full_null.csv | time tippecanoe -P -Z 11 -z 14 -d 14 --detect-shared-borders -x block_fips --coalesce-smallest-as-needed -l block_null_z11 -f -o mbtiles/block_null_z11.mbtiles 2>&1 | tee ./block_null_z11.log

tile-join -f -o mbtiles/block_null.mbtiles mbtiles/block_null_z10.mbtiles mbtiles/block_null_z11.mbtiles
```
### Provider footprints
We do not need to join data to the provider footprints since they require only the ID of the provider (hoconum) and geometry.

The following code was used to create the provider geojsons but ***this should be replaced with geopandas.***  In addition, this code breaks out the different geojsons and tilesets by technology.  ***We probably need to change this to breakout by hoconum*** so that it will be clear which tileset should be added to the map when a provider is chosen by the end user.
```
ogr2ogr -f GeoJSON '/Users/steven/Data/No_TimeMachine/SOBBM/Provider_test/providers_exsat_test.geojson' PG:"host=gisp-proc-wcb-pg-int.cfddrd5nduv6.us-west-2.rds.amazonaws.com user=wcb_srosenberg dbname=wcb_internal" -sql "SELECT left(block_fips,5) as county_fips, hoconum, ST_Union(geom) as geom FROM (SELECT block_fips, hoconum FROM f477.fbd_jun2016_10feb17 WHERE consumer=1 AND transtech!=60 GROUP BY block_fips, hoconum) foo LEFT JOIN census2010.block_us ON geoid10=block_fips GROUP BY county_fips, hoconum" -progress
ogr2ogr -f GeoJSON '/Users/steven/Data/No_TimeMachine/SOBBM/Provider_test/providers_sat_test.geojson' PG:"host=gisp-proc-wcb-pg-int.cfddrd5nduv6.us-west-2.rds.amazonaws.com user=wcb_srosenberg dbname=wcb_internal" -sql "SELECT left(block_fips,5) as county_fips, hoconum, ST_Union(geom) as geom FROM (SELECT block_fips, hoconum FROM f477.fbd_jun2016_10feb17 WHERE consumer=1 AND transtech=60 GROUP BY block_fips, hoconum) foo LEFT JOIN census2010.block_us ON geoid10=block_fips GROUP BY county_fips, hoconum" -progress
```

The following code was used to create the provider tilesets; we may need to revisit the simplification that `tippecanoe` does with these settings (and perhaps add `--coalesce-smallest-as-needed`).
```
tippecanoe -o '/Volumes/Steven.Rosenberg/My Documents/_WCB/SOBBM/provider_test/provider_exsat_test.mbtiles' -P -x county_fips -z 13 -Z 2 -f -d 14 --coalesce -l provider_exsat_test '/Users/steven/Data/No_TimeMachine/SOBBM/Provider_test/providers_exsat_test.geojson'
tippecanoe -o '/Volumes/Steven.Rosenberg/My Documents/_WCB/SOBBM/provider_test/provider_sat_test.mbtiles' -P -z 13 -Z 2 -f -d 14 --coalesce -l provider_sat_test '/Users/steven/Data/No_TimeMachine/SOBBM/Provider_test/providers_sat_test.geojson'
```
## Background and Reference

### Using `join-data` to join geometry and tabular data
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

### `tippecanoe` settings to create tilesets
There are a number of options in `tippecanoe` designed to ensure that each tile size remains below the 500k limit. The difficulty is in finding a combination of those options that provides enough reduction in tile size without creating odd visual artifacts or gaps (e.g., all the --drop flags will open up gaps in the data and can’t be used for these data).

#### Removing unneeded data
As noted above, the tilesize includes the data payload, so it's important to drop any unnecessary fields using the `-x` or `-X` flags on `tippecanoe`.  Exactly what fields need to be excluded using `-x` (or included using `-y`) depends on the original source for the geojson and what fields it includes beyond the FIPS code and geometry.  For broadband data, we only want to include the 315 broadband values that matter for county and tract data, and the 315 broadband values plus the h2only_undev field.  For the state, county, CBSA, CD, CDP and tribal areas, we need the ID associated with each geography's geometry; and for the provider map, we need the hoconum.  **Any other data fields should be dropped.**

#### Reducing tile size by removing geometry data...without making it look bad.
By default, `tippecanoe` reduces the level of detail for individual tiles when a tile is larger than 500k to try and reduce the tile size. It will continue to reduce the level of detail until it fits, or it gets to a detail level of 7 (a tile of 128 pixels across) and fails. This reduction in detail can cause odd visual artifacts, especially at low levels of detail. The reduction in size comes from greater simplification (see description of simplification below).

Given the problems with the reduction in detail, and to avoid opening up gaps using the various `-drop` flags, this process relies on `--coalesce` and `--coalesce-smallest-as-needed` to reduce the size of tiles.

The `--coalesce` flag aggregates together geometries with the same field values (in our case, across all 315/316 variables) that are adjacent, basically performing the equivalent of ST_Union in PostGIS and dissolving the internal boundaries. This is among the reasons why it’s important to drop the FIPS code from the data set – since each geometry has a unique value, the FIPS field would otherwise prevent `--coalesce` from having any effect (the other reason is that it’s more data payload which requires even greater savings from reducing the geometry). Using `--coalesce` means there is less use of the `--coalesce-smallest-as-needed` flag. The downside to using `--coalesce` is that you lose the boundaries between blocks and can create relatively large blobs. Note that `--coalesce` is likely to have little effect on tract and county data since those are floating point numbers (i.e., they have more variation in value).

The `--coalesce-smallest-as-needed` flag simply identifies the smallest-area geometries and merges them into their neighbors. Right now, the combined geometry has the data values of the larger geometry. That can and does change the values for some areas and so can change the appearance. Thus the use of `--coalesce` to minimize the number of blocks that might change their value.

***Right now, `--coalesce-smallest-as-needed` is creating unexpected outputs in zoom 10 (where it is critical)***

##### How `tippecanoe` simplifies geometry data
The distance represented by each pixel depends on the zoom level and the number of pixels per tile.

Zoom level 0 incorporates the whole earth, and each zoom level higher represents a 2x gain in resolution in x and in y. Zoom level 1 is a 2x2 grid covering the earth; zoom level 2 is a 4x4 grid. So the degrees of lat/lon covered by each tile is ~360/(2^(zoom level).

The number of pixels in each tile is controlled by the level of detail – the number of pixels is 2^detail level. So, for the default detail 12, there are 2^12 pixels or 4096 pixels. Taken with the zoom level, the resolution of each pixel is ~360/(2^(zoom level + detail)) in degrees. Taking ~0.00000274 feet per degree (at the equator), and you get the distance per pixel, which depends on zoom level and detail level. So, e.g., at zoom level 10 and detail level 12, each pixel is ~(360/(2^(10+12))/.00000274) = 32 feet.

##### Simplification

By default, the resolution per pixel is the distance that Tippecanoe uses in applying the Douglas-Peucker simplification (this is the same simplification in ST_Simplify and the default algorithm used in ArcGIS Pro). The `-S` command overrides this distance and provides a value to multiply that distance by for the simplification; i.e., a value of `-S 8` would use a distance of 8x32 feet at zoom level 10 and detail level 12.

By default, the simplification modifies each polygon independently. This means that at higher simplification levels, gaps can open up between polygons, which is a problem for data like blocks, tracts and counties which should have complete coverage. The `--detect-shared-borders` flag is meant to prevent this from happening. [it appears there’s a bug in the implementation so that when detect-shared-borders is set and the level of detail decreases gaps will still open up. See https://github.com/mapbox/tippecanoe/issues/482].

#### Overzoom to avoid creating higher-zoom data than needed

The `-d` flag sets the detail level at the highest zoom (the default is 12). By setting `-d 14`, you are adding 4x as much resolution to the highest zoom level (set with the `-z` option), which means that as you zoom in past the highest zoom level, you have the resolution as if you had created two additional zoom levels. The advantage to using the `-d` option is that the resulting mbtiles file is much smaller (and is created much faster) than creating two additional zoom levels.

### Using `tile-join` to combine tilesets
We can reduce the number of tilesets by combining two tilesets into one in some instances using `tile-join`.  The 500k-per-tile limit applies when using `tile-join`.  So where we have tilesets that are at different zoom levels -- i.e., where overlaying two different tilesets won't create a tile that exceeds 500k -- we can combine tilesets.  This works for county data and block data, which have tilesets at different zoom levels; it also seems to work for tract and block data at zoom 9.  However, it will not work for tract data at zoom levels 5-8 since the tileset for each speed can approach 500k.  

The syntax to combine two tilesets into one is `tile-join -f -o output.mbtiles input1.mbtiles input2.mbtiles` This puts each input tileset into a layer of the output tileset (we will investigate whether we can combine into one layer).
