# Processing a bunch of data
David Hood

## Day 23 of the 30 day map challenge 2025

The prompt for this day is “Process”, and one of the examples for the
prompt is a tutorial explaining how a previous map was made. And I
thought it would be useful for me to write up the handling of data for
the day 16 map - cell - as it involved some DuckDB steps that I hadn’t
used before and learning more about the NZTM2000 Coordinate System. So
this is a note to self and others.

For the theme of cells, I was looking for something geographic to
aggregate into cell blocks, and found a point cloud of LIDAR data for
Dunedin and Mosgiel (New Zealand) from 2021. There is an average of
13.64 points per m² for the surveyed areas.

https://data.linz.govt.nz/layer/d3W3yfd9hbtmxtQ/otago-dunedin-and-mosgiel-lidar-pointcloud-2021/

As the point cloud contains 720,421,588 points, I assumed from the
outset I would be using DuckDB to process the data and didn’t really
look at other options. Because I am very R comfortable, I am using the
duckplyr R interface to save thinking time.

``` r
library(duckplyr)
```

But for reducing the number of points, I used the export region
selection tools to just select the Dunedin data reducing the size of the
data to a 3.24 GB zipped download which decompressed to a 13.52 GB csv
file (I selected csv as the download format option). Which is still very
much a size sensible to use DuckDB.

The data CRS I chose as part of the export options was NZGD2000 / New
Zealand Transverse Mercator 2000 + NZVD2016 height which is a very
common coordinate system for New Zealand only material. One unit of
NZTM2000 is pretty-much one metre (it varies a little around the country
with the whole issue of applying a grid to an oblate spheroid).

For a cells map, I need to decide on the fill content of the cells. Of
the two interesting options in the data, the elevation and a numeric
code for a processed estimate of what the point was, the type of point
seemed more interesting to aggregate.

I also want to save out an aggregated file, so as well as providing the
input path, I am giving an output path to save that to. Also, because I
read that it is better to provide the data types of the column, and all
6 columns are numeric, I am setting that.

``` r
in_csv      <- "/path/to/input.csv"
out_csv     <- "/path/to/output.csv"
type_override <- list(c("DOUBLE", "DOUBLE", "DOUBLE","DOUBLE", "DOUBLE", "DOUBLE"))
```

So, let’s write some code reading the data, but keeping it in DuckDB for
as long as possible. By setting it to “stingy” it generates an error for
my attention if it is going to have to load things into R’s memory.

``` r
df <- read_csv_duckdb(
  path     = in_csv,
  prudence = "stingy",                      # never auto-materialize to R memory
  options  = list(header = TRUE, types = type_override)
)
```

Now I want to convert the NZTM2000 coordinates to 10 metre blocks. One
thing I found at this point, due to helpful Stingy generated errors, is
that using R’s aggregation functions such as floor would cause the data
to load into memory. But reading up, one can draw on DuckDb’s own
functions to keep things abstract. In particular DuckDBs floor function
was used to chunk things into grids. I also checked, code not shown, the
minimum X and Y values to find where to grid from (not that I really
needed to do this, I could just have aggregated the data as is, but it
made me happier to start from 0).

Because it is not quite 1 NZTM2000 unit is a metre, I also
(pedantically) looked up what unit size I should use to represent 10
metres on the ground in Dunedin. It would still have worked as a map
dividing by 10 to make chunks, but I felt better knowing I was dividing
by 9.9993867587 to be as close as possible to 10 metres on the ground.

``` r
df_calc <- df %>%
  transmute(
    Xnew = dd$floor( (X - 1404159) / 9.9993867587 ),
    Ynew = dd$floor( (Y - 4912079) / 9.9993867587 ),
    Z = Classification
  )
```

So we now have three variables, A grid X coordinate and grid Y
coordinate shared by on average around 136 points in the 10 square
metres. And a Z “classification” coordinate with the processed terrain
type of the point. So we count up each combination of classifications
and grid points to see how common they are.

``` r
counts <- df_calc %>%
  count(Xnew, Ynew, Z)
```

This implicitly creates a new variable called n as the sample size of
the combination.

Now, to find the commonest classification for each grid, I took the
database approach of finding the maximum n for each grid, then joining
that information back to the grid entries, and selecting which entries
match the maximum entry for the grid. A very databasey way of reducing
the dataset to the maximum values.

``` r
max_per_xy <- counts %>%
  summarise(.by = c(Xnew, Ynew),
            max_n = max(n))

result <- counts %>%
  inner_join(max_per_xy, by = c("Xnew", "Ynew")) %>%
  filter(n == max_n) %>%
  select(Xnew, Ynew, Z)
```

This can result in tied maximal values for the same grid being present,
but I don’t care in this case because all it will do is overplot one
grid cell with another. And that seems a reasonable way to resolve the
tie in a casual map.

Finally (in the data processing part) I am writing out a csv of results.
Keeping the whole process in duckDB, with the useful side effect of
being able to see how big the resulting csv is by having a look at it in
the file system

``` r
compute_csv(result, out_csv)
```

The last data adjustment was that 10 classification categories was just
too many to show with different colours, so I reduced the set by
combining some of the classifications where it made sense for me.
Combining the low, medium, and high vegetation to a single “Vegetation”
and the various forms of unclassified entries to “Unclassified/Noise”.

Then, to actually make the map, I made a geom_point graph with the X and
Y coordinates constrained to the same size.
