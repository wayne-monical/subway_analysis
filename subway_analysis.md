Subway Analysis
================
Wayne Monical
2024-10-01

``` r
library(tidyverse)
```

    ## Warning: package 'tidyverse' was built under R version 4.4.2

    ## Warning: package 'ggplot2' was built under R version 4.4.2

    ## Warning: package 'dplyr' was built under R version 4.4.2

    ## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ## ✔ dplyr     1.1.4     ✔ readr     2.1.5
    ## ✔ forcats   1.0.0     ✔ stringr   1.5.1
    ## ✔ ggplot2   3.5.1     ✔ tibble    3.2.1
    ## ✔ lubridate 1.9.3     ✔ tidyr     1.3.1
    ## ✔ purrr     1.0.2     
    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()
    ## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

``` r
library(readxl)
```

    ## Warning: package 'readxl' was built under R version 4.4.2

## Subway Data

### Data Cleaning

We begin by importing and cleaning the subway data set. This data set
contains information on the organization, location, and amenities in the
New York city subway system. The data is tidy. The data is consistent,
i.e. each row of the data frame contains information on one subway
entrance or exit. It is structured, i.e. every variable, such as
station, line, and latitude, has its own column. Finally, all values
have their own cell, i.e. there is no value concatenation.

``` r
subway = read_csv(
  "data/NYC_Transit_Subway_Entrance_And_Exit_Data.csv") |> 
  janitor::clean_names()
```

    ## Rows: 1868 Columns: 32
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (22): Division, Line, Station Name, Route1, Route2, Route3, Route4, Rout...
    ## dbl  (8): Station Latitude, Station Longitude, Route8, Route9, Route10, Rout...
    ## lgl  (2): ADA, Free Crossover
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
subway_cols = c("line", "station_name", "station_latitude", "station_longitude",
                   "route1", "route2", "route3", "route4", "route5", "route6", "route7", 
                   "route8", "route9", "route10", "route11", "entrance_type", "entry",
                   "exit_only", "vending", "ada",    "ada_notes", "entrance_latitude",
                   "entrance_longitude")

subway = subway |> select(all_of(subway_cols))
```

We see here that `entry`, `exit_only`, and `vending` are character
vectors, and `ada` is a logical vector.

``` r
logical_vars = c('entry', 'exit_only', 'vending', 'ada')

subway |> 
  select(all_of(logical_vars)) |> 
  summary()
```

    ##     entry            exit_only           vending             ada         
    ##  Length:1868        Length:1868        Length:1868        Mode :logical  
    ##  Class :character   Class :character   Class :character   FALSE:1400     
    ##  Mode  :character   Mode  :character   Mode  :character   TRUE :468

Inspecting the unique values of each character, we can find the
character strings that correspond to each logical value.

``` r
char_vars = c('entry', 'exit_only', 'vending')

for(var in char_vars){
  print(var)
  
  subway |> 
    pull(var) |> 
    unique() |> 
    print()
}
```

    ## [1] "entry"
    ## [1] "YES" "NO" 
    ## [1] "exit_only"
    ## [1] NA    "Yes"
    ## [1] "vending"
    ## [1] "YES" "NO"

Using the `mutate()` function, we can reassign these variables to be
logical based on the character strings that we found.

``` r
subway = subway |> 
  mutate(
    entry = entry == 'YES',
    exit_only = !is.na(exit_only),
    vending = vending == 'YES')
```

There are 465 distinct subway stations.

``` r
subway |> select('station_name', 'line') |> distinct() |> nrow()
```

    ## [1] 465

We must also standardize the route variables as characters.

``` r
subway = subway |> 
  mutate(
    route1 = as.character(route1),
    route2 = as.character(route2),
    route3 = as.character(route3),
    route4 = as.character(route4),
    route5 = as.character(route5),
    route6 = as.character(route6),
    route7 = as.character(route7),
    route8 = as.character(route8),
    route9 = as.character(route9),
    route10 = as.character(route10),
    route11 = as.character(route11),
  )
```

There are 84 ADA compliant subway stations.

``` r
subway |> filter(ada) |> select('station_name', 'line') |> distinct() |> nrow()
```

    ## [1] 84

73% of station entrances and exits without vending allow entrance.

``` r
subway |> filter(!vending) |> pull(exit_only) |> summary()
```

    ##    Mode   FALSE    TRUE 
    ## logical     133      50

``` r
133 / (50 + 133)
```

    ## [1] 0.726776

### Pivoting

Here, we pivot the route variables to a longer format and drop the rows
that do not have a route.

``` r
subway= subway |> 
  pivot_longer(
    route1:route11,
    names_to = 'route_number',
    values_to = 'route'
  ) |> 
  filter(!is.na(route))
```

Using the code below, we find that 17 of the 43 of the stations on the
A, 39%, are ADA compliant.

``` r
subway |> 
  filter(route == 'A') |> 
  select('line', 'station_name', 'ada') |>
  distinct() |> 
  pull(ada) |>
  summary()
```

    ##    Mode   FALSE    TRUE 
    ## logical      43      17

``` r
107 / (166 + 107)
```

    ## [1] 0.3919414

## Mr Trash Wheel

Mr Trash Wheel is a machine that picks trash from the water with a large
wheel. We will now analyze how much trash and what kind of trash he
picks. We begin by reading in the 2024 Mr Trash Wheel sheet using the
`read_excel` function. We have specified the excel sheet and the data
range, and we have rounded the number of sports balls and set it to
integer values. We also transform the year variable into an integer, so
that when we combine this data set with Gwynnda Trash Wheel’s data set,
we are able to smoothly bind the rows. We have omitted the totaling rows
from the original excel sheet, since they do not contain
dumpster-specific data.

``` r
mr_trash = 
  read_excel(
    path = 'data/202409 Trash Wheel Collection Data.xlsx',
    sheet = 'Mr. Trash Wheel',
    range = 'A2:N653') |> 
  janitor::clean_names() |> 
  mutate(
    sports_balls = sports_balls |> round() |> as.integer(),
    year = year |> as.integer())
```

Using the same code structure, we can import and clean Gwynnda Trash
Wheel’s data to create a single tidy data set. This data set is tidy
because each row corresponds to the data from a single dumpster, each
column corresponds to a single variable, and each cell contains a single
value.

``` r
gwyndda_trash = 
  read_excel(
    path = 'data/202409 Trash Wheel Collection Data.xlsx',
    sheet = 'Gwynnda Trash Wheel',
    range = 'A2:K265') |> 
  janitor::clean_names()
```

Adding a variable for `trash_wheel`, we can combine Mr Trash Wheel’s
data with Gwynnda’s data with the `bind_rows` function, stacking the
rows on top of each other. Note that there are some differences between
the two data sets. For example, Gwynnda Trash Wheel has no
`sports_balls` variable, so this variable is `NA` in Gwynnda’s rows.

``` r
mr_trash = mr_trash |> mutate(trash_wheel = 'mr_trash')

gwyndda_trash = gwyndda_trash |> mutate(trash_wheel = 'gwynnda')

trash = bind_rows(mr_trash, gwyndda_trash)
```

With this tidy data set, we can use `dyplr` functions to answer any
concrete question about the data. For example, from 2014 to 2024,
Mr. Trash Wheel collected a total of 2091 tons of trash.

``` r
trash |> filter(trash_wheel == 'mr_trash') |> pull(weight_tons) |> sum()
```

    ## [1] 2091.18

By filtering for bins collected by Gwynnda in June of 2022, we can find
that she collected a total of 1.812^{4} cigarette butts in that month.

``` r
trash |> 
  filter(
    trash_wheel == 'gwynnda',
    year == 2022,
    month == 'June') |> 
  pull(cigarette_butts) |> sum()
```

    ## [1] 18120

## Great British Baking Show

### Data Carpentry

The first step is to import the data and look at it. Any non-tidy data
can be corrected at this stage.

1)  The `bakers` and `bakes` data is structured as expected, but the
    `results` data is not. Examining the csv file, we can see that the
    data frame begins on the third row. Specifying the argument
    `skip = 2` instructs the `read_csv()` function to skip the first two
    rows of the csv file, and therefore read the data as we expect.

2)  The data will be joined by the first names of the bakers, their
    series, and their episode. Therefore we will create the variable for
    first name in the `bakers` data frame by extracting the first word
    from the `baker_name` variable.

3)  The `bakes` data frame has the column `signature_bakes` with the
    value “N/A”, which should be treated as `NA`. We can specify this as
    a missing value in the `reaed_csv` function.

``` r
bakers =
  read_csv('data/gbb_datasets/bakers.csv') |> 
  janitor::clean_names() |> 
  mutate(baker = stringr::word(baker_name, 1))
```

    ## Rows: 120 Columns: 5
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (3): Baker Name, Baker Occupation, Hometown
    ## dbl (2): Series, Baker Age
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
bakes =
  read_csv(
    'data/gbb_datasets/bakes.csv',
    na = c('N/A')) |> 
  janitor::clean_names()
```

    ## Rows: 548 Columns: 5
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (3): Baker, Signature Bake, Show Stopper
    ## dbl (2): Series, Episode
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
results =
  read_csv('data/gbb_datasets/results.csv', skip = 2) |> 
  janitor::clean_names()
```

    ## Rows: 1136 Columns: 5
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (2): baker, result
    ## dbl (3): series, episode, technical
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

The second step is to check for completeness. We use the code below to
search for any discrepancy between the `bakers` and the `results` data
frame. We find that there is one missing name in each data set, namely
“Jo” and “Joanne”. Upon further research, we find that both of these
names occur in series 2, and are very likely the same person.

``` r
anti_join(bakers, results) |> pull(baker) |> unique()
```

    ## Joining with `by = join_by(series, baker)`

    ## [1] "Jo"

``` r
anti_join(results, bakers) |> pull(baker) |> unique()
```

    ## Joining with `by = join_by(series, baker)`

    ## [1] "Joanne"

Here we change all instances of “Jo” to Joanne. So now the `bakers` and
`results` data frames agree.

``` r
bakers = 
  bakers |> 
  mutate(baker = ifelse(baker == "Jo", "Joanne", baker))

anti_join(bakers, results) |> pull(baker) |> unique()
```

    ## Joining with `by = join_by(series, baker)`

    ## character(0)

``` r
anti_join(results, bakers) |> pull(baker) |> unique()
```

    ## Joining with `by = join_by(series, baker)`

    ## character(0)

We find and correct the same Joanne. issue in the `bakes` data set

``` r
anti_join(bakes, bakers) |> pull(baker) |> unique()
```

    ## Joining with `by = join_by(series, baker)`

    ## [1] "\"Jo\""

``` r
bakes = 
  bakes |> 
  mutate(baker = ifelse(baker == "\"Jo\"", "Joanne", baker))
```

However, there are still 24 names that are missing from `bakes`, all of
which are from series 9 and 10. Inspecting `bakes`, we see that the max
series it has is up until season 8, explaining the difference.

``` r
anti_join(bakes, bakers) |> pull(baker) |> unique()
```

    ## Joining with `by = join_by(series, baker)`

    ## character(0)

``` r
anti_join(bakers, bakes) |> 
  select(series, baker) |> 
  arrange(series, baker)
```

    ## Joining with `by = join_by(series, baker)`

    ## # A tibble: 25 × 2
    ##    series baker  
    ##     <dbl> <chr>  
    ##  1      9 Antony 
    ##  2      9 Briony 
    ##  3      9 Dan    
    ##  4      9 Imelda 
    ##  5      9 Jon    
    ##  6      9 Karen  
    ##  7      9 Kim-Joy
    ##  8      9 Luke   
    ##  9      9 Manon  
    ## 10      9 Rahul  
    ## # ℹ 15 more rows

``` r
bakes |> pull(series) |> max()
```

    ## [1] 8

### Merging Data

Here we merge the data with left joins, since there are missing data
points from `bakes`. In the code output, we see that we have
successfully joined on our intended columns. The data set is tidy. Each
row represents an individual contestants performance on a single episode
of a single series. If a contestant does not compete in an episode since
they are eliminated, their `result` variable is `NA`, so we may also
drop these rows. We also reorder the rows and columns for clarity.
Finally, we save this data frame as a csv file.

``` r
gbb = 
  bakers |> 
  left_join(results) |> 
  left_join(bakes) |> 
  filter(!is.na(result)) |> 
  select(series, baker_name, baker, baker_age, baker_occupation, 
         hometown, episode, result, technical, signature_bake, 
         show_stopper) |> 
  arrange(series, baker_name, episode, result)
```

    ## Joining with `by = join_by(series, baker)`
    ## Joining with `by = join_by(series, baker, episode)`

``` r
gbb |> write.csv('data/gbb_datasets/great_british_bake_off.csv')
```

### Star Baker Table

In the code below, we filter for episodes in seasons 5 through 10 and
the name of the star baker in each episode. I included the winner of the
season in this table, because the season winner is the star baker of the
final episode in each season. I arranged the table in order of series
and then episode. In season five, we see that Richard Burr was star
baker in five of the ten episodes, but failed to secure the win at the
end of the season, which is surprising.

``` r
gbb |> 
  filter(
  series >= 5,
  series <= 10,
  result %in% c('STAR BAKER', 'WINNER')
  ) |> 
  select(
    series, baker_name, episode, result
  ) |> 
  arrange(series, episode)
```

    ## # A tibble: 60 × 4
    ##    series baker_name        episode result    
    ##     <dbl> <chr>               <dbl> <chr>     
    ##  1      5 Nancy Birtwhistle       1 STAR BAKER
    ##  2      5 Richard Burr            2 STAR BAKER
    ##  3      5 Luis Troyano            3 STAR BAKER
    ##  4      5 Richard Burr            4 STAR BAKER
    ##  5      5 Kate Henry              5 STAR BAKER
    ##  6      5 Chetna Makan            6 STAR BAKER
    ##  7      5 Richard Burr            7 STAR BAKER
    ##  8      5 Richard Burr            8 STAR BAKER
    ##  9      5 Richard Burr            9 STAR BAKER
    ## 10      5 Nancy Birtwhistle      10 WINNER    
    ## # ℹ 50 more rows

### Viewership

Here we import, clean, and analyze the viewership data from the Great
British Bake Off. We see that the data is structured in the wide format,
with each row containing all ratings of each episode number across ten
seasons. In order to make this data tidy we will pivot the data so that
each row is a single viewership number of a single episode. We will drop
the missing values, since they correspond to non-existent episodes in
the first several seasons. We will arrange the rows and columns for
clarity.

``` r
viewers =
  read_csv('data/gbb_datasets/viewers.csv') |> 
  janitor::clean_names() |> 
  pivot_longer(
    series_1:series_10,
    names_to = 'series',
    values_to = 'viewership'
  ) |> drop_na() |> 
  select(series, episode, viewership) |> 
  arrange(series, episode, viewership)
```

    ## Rows: 10 Columns: 11
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## dbl (11): Episode, Series 1, Series 2, Series 3, Series 4, Series 5, Series ...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

Here are the first ten rows of the `viewer` dataframe.

``` r
viewers |> head(10)
```

    ## # A tibble: 10 × 3
    ##    series    episode viewership
    ##    <chr>       <dbl>      <dbl>
    ##  1 series_1        1       2.24
    ##  2 series_1        2       3   
    ##  3 series_1        3       3   
    ##  4 series_1        4       2.6 
    ##  5 series_1        5       3.03
    ##  6 series_1        6       2.75
    ##  7 series_10       1       9.62
    ##  8 series_10       2       9.38
    ##  9 series_10       3       8.94
    ## 10 series_10       4       8.96

The average viewership by season is given below.

``` r
viewers |> 
  group_by(series) |> 
  summarise(mean_views = round(mean(viewership), 2))
```

    ## # A tibble: 10 × 2
    ##    series    mean_views
    ##    <chr>          <dbl>
    ##  1 series_1        2.77
    ##  2 series_10       9.24
    ##  3 series_2        3.95
    ##  4 series_3        5   
    ##  5 series_4        7.35
    ##  6 series_5       10.0 
    ##  7 series_6       12.3 
    ##  8 series_7       13.6 
    ##  9 series_8        9.02
    ## 10 series_9        9.3
