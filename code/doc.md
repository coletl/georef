# Approximate Georeferencing
J. Andrew Harris  
Cole Tanigawa-Lau  
<style>
body {
text-align: justify}
</style>



# Motivation

Vast improvements in processing power have opened the potential of spatial data analysis to the average social science researcher, laying the foundations for empirical work that has incorporated increasingly complex analyses of spatial relationships. Both the public and private sectors in many highly developed countries have advanced their spatial data capabilities in parallel with technological innovations, producing high-quality GIS products and services to meet the rising demand. In the political science literature, these data have yielded numerous high-powered, micro-level studies of behavior and political phenomena.[^micro_examples] 

Many developing countries do not have such robust support for researchers hoping to conduct spatial analyses. Development institutions have produced high-resolution raster interpolations on a near-global scale for quantities like population density, literacy, and mortality; however, key political covariates exist only at much higher levels of aggregation. Government-issued shapefiles for administrative boundaries tend to be of much lower quality and are subject to change over time, with or without a formal delimitation procedure. Accurate geographic coordinates are simply unavailable for many types of point data.[^gmaps] Even if a local GIS industry were to develop quickly or governments were to invest heavily in improving their GIS files, many micro-level studies would remain impossible to replicate in a developing context, since data like the lists of individuals' addresses contained in U.S. voter registers are not even collected in most countries.

The dearth of readily available, high-quality spatial data in the developing world, we believe, dramatically increases the relative demand for such a product. These data are not wholly inaccessible. With experience admittedly limited to Kenya, we would conjecture that spatial data in fact do exist for many developing countries. The problem is that these data are distributed sparsely across various people, departments, and organizations to the extent that significant time and personal effort are required to collect the data files even before any matching processes begin. As long as researchers cannot simply download these data in prepackaged form, though, they remain an untapped resource for those willing to put in the hours required to link the mass of geocoded points available from an inconvenient number of sources to the lists of schools, health clinics, markets, administrative offices, and polling centers that social scientists hope to study.

The remainder of this document discusses in detail our process of matching location names to geographic points.
<!-- [Background] relates our methods to standardized strategies of record linkage. -->
[Example Data] describes the sample of polling stations matched in this document and the various points and polygons required for the linkage process. [Georeferencing] explains the techniques we've found helpful in our matching method.
Talk about de-duplication?
[Imputing points] discusses our process of stochastically imputing point coordinates for polling stations we could not accurately georeference.

We assume of the reader an intermediate-level understanding of the `R` programming langauge and basic familiarity with the S4 spatial data classes defined in the `sp` package [@base;@sp]. Many processes in this document are run in parallel on six cores. All operations are performed on a server running R version 3.4.0 (64-bit) under Ubuntu 14. Reproducing the results on a machine with less than 32GB of RAM or the 32-bit version may require adjusting the code.

[^micro_examples]: Cite examples here. Footnote okay, or do we need to talk about their work with in-text citations?
[^gmaps]: As an example, we suggest using Google Maps, an excellent geocoding tool, to search for a point in the United States of America using very little information (e.g., ). Searching for a fairly comparable point in Kenya while supplying much more data (e.g., ).

<!-- # Background -->
<!-- ## Literature on record linkage -->
<!-- Christen [@Christen2012] offers one of the very few contemporary surveys of the record linkage literature, describing complex recent advances in generalized matching techniques . . . -->


<!-- ## Applications in the social sciences -->

# Example Data
We demonstrate our methods using samples of the polling stations file and the geocoded points. We also rely on polygon shapefiles detailing several levels of Kenyan administrative boundaries at different points in time. Finally, we use a 2010 population density raster for Kenya from [WorldPop](www.worldpop.org) [@Linard2012]. All of these data are available to download [here](https://drive.google.com/open?id=0B8K1PQKTPN42V19EcGIzUHhJb3c).


```r
# Load packages
library(sp)
library(maptools)
library(rgeos)
library(rgdal)
library(raster)
library(spatstat)

library(parallel)
library(doMC)
library(foreach)

library(stringdist)
library(stringr)
library(dplyr)

# Load user-defined functions
source("code/functions.R")
```

## Points
We use two sets of point-specific data. First, the polling stations file, `data/ps_samp.rds`, is a sample of polling stations scraped from voter registration and electoral results documents produced in relation to the 1997, 2002, 2007, 2010, and 2013 elections in Kenya as well as the 2005 constitutional referendum. These government files provide not only the name of the active polling station in that year (e.g., "MURKISIAN PRIMARY SCHOOL") but also important information on what administrative units govern the polling station. These regionally identifying variables allow us to index to a geographic area, or "block", within which we roughly restrict our searches. (More on this below.)

Second, we have a mass of geocoded "match points" in Kenya obtained from various local, foreign, public, and private sources. A sample of these is stored in `data/mp_samp.rds`. The accuracy of the coordinates and the consistency of formatting in the name strings vary by source. Each "match point" represents a potential location for a given polling station, but the match points file is, in reality, neither complete nor unambiguous. Therefore, while the procedure outlined here links these two files and yields a version of the polling stations file with geographic coordinates attached to each record, some of these coordinates must be imputed rather than matched to.


```r
# Read points data
ps <- readRDS("data/ps_samp.rds")
mp <- readRDS("data/mp_samp.rds")
```

## Polygons

The polygon files included with this document represent several administrative levels. Redistricting and revisions in data quality have produced files that record changes in boundaries over time. In roughly decreasing order of scale, the units are as follows:

1. pre-2010 (or "old") constituencies, 
2. post-2010 constituencies,
3. post-2010 constituency assembly wards ("wards" or "CAWs"),
4. pre-2010 wards (or "old wards"), 
5. locations,
6. sublocations.


```r
# Read polygons data
const_old <- readRDS("data/const_old.rds")
const13 <- readRDS("data/const13.rds")
# caw <- readRDS("data/caw.rds")
# caw_old <- readRDS("data/caw_old.rds")
sl <- readRDS("data/sl.rds")
```

Other regionally identifying information for polling stations, such as county, division, and province, are sometimes available, but they denote units that are too large to be helpful in narrowing down our searches.

Because the boundaries of these geographic units have changed over time, and the Kenyan government has not always provided these data at a high quality, we cannot perfectly and consistently map polling stations to units like the county, constituency, ward, location, or sublocation. This difficulty is particularly pronounced along the borders of administrative units, where anecdotal evidence informs us that what the central Kenyan government may understand and treat as dividing lines are not as clear-cut to chiefs and other local administrators on the ground. Therefore, the borders delineated by the polygon shapefiles are treated as only soft boundaries on point locations.

## Assessing data quality

Understanding the quality of the polygon shapefiles informs us, if roughly, of how much error we should expect at the borders of administrative units. We've found two attributes of shapefiles to be most informative with only a quick study of the data: boundary detail and border alignment.

### Boundary detail
First, the level of detail provided at the boundaries of shapefile features indicates the amount of time, money, and effort put into creating the data. Building shapefiles that accurately represent the geography on the ground is labor intensive. Shortcuts, just like lower-resolution raster files, produce unrealistically rectangular boundaries where the land's natural cuts and curves were too costly to map. These differences are most conspicuous along shorelines.

Below, we plot the overlay of three constituencies bordering Lake Victoria as represented in two different shapefiles.
![](doc_files/figure-html/bound_plot-1.png)<!-- -->

Such exploratory work is most easily performed in an interactive GUI like QGIS or ArcGIS, but even this simple plot demonstrates the vast differences in detail offered by the two files: a clear coastline, rather than blockish figures, and distinct islands.

### Border alignment

+ Need examples

**A note on CRS (using UTM)**


# Georeferencing
We now outline our methods of finding the closest geographic match for polling stations in Kenya. It is worth noting now that our objective is neither to find the closest string match nor to find as many schools or health clinics as possible. Rather, we aim to minimize geographic error, even at the cost of finding what is certainly the wrong point.

## Preprocessing

It would be difficult to overstate the importance of the preprocessing stage. Time spent developing the consistency of the data early in the matching procedure can lead to enormous gains in accuracy down the road. Perhaps more importantly, this phase develops a familiarity with the data's idiosyncracies that becomes invaluable in developing strategies tailored to a specific data set or goal. By its nature, however, preprocessing requires techniques that may vary widely depending on whatever quirks of the unmatched data require standardization. We therefore begin with data that has already gone through some cleaning. We describe only a few preprocessing steps that are helpful in the most general cases, omitting any discussion of the more idiosyncratic adjustments we made.

### Truncating strings

<!-- Looking at a small subset of the polling station names, we see that many of the polling stations are located at schools. -->
<!-- ```{r print_ps_samp} -->
<!-- set.seed(10) -->

<!-- sample_n(ps@data, 5) %>% dplyr::select(starts_with("name")) -->
<!-- ``` -->

Polling stations were sometimes recorded in the pattern `ALPHA PR. SCH`, but variations including `ALPHA PRY SCH`, `ALPHA PRIMARY SCHOOL`, and `ALPHA PR.SCH` also exist. This lack of standardization is common for all types of polling station locaitons. One approach would be to standardize the abbreviation of `PRIMARY` and `SCHOOL` by replacing all permutations with a single string. However, the algorithm we use for fuzzy matching not only inspects differences between strings but also normalizes that edit distance by the number of characters in each string. In the example below, we return two different string distance scores given inputs of essentially the same information.


```r
stringdist("ALPHA PRIMARY SCHOOL", "BETA PRIMARY SCHOOL", method = "jw", p = 0.15)
```

```
## [1] 0.1817982
```

```r
stringdist("ALPHA PR. SCH", "BETA PR. SCH", method = "jw", p = 0.15)
```

```
## [1] 0.2229345
```
For our purposes, it is more important to match polling stations to the correct geographic area, rather than matching primary schools strictly to primary schools.[^alpha_beta] We can therefore remove the nonessential information and store it separately. We do this by cataloging the most common types of polling stations, then truncating the strings to separate these labels from the information that identifies the town, village, or area the polling station is located in. The result is a truncated polling station name (e.g., `ALPHA`) with classifying information (e.g., `SCH` and `PRI`) in separate columns. Thus, when calculating the similarity of two strings (e.g.,  `ALPHA` and `BETA`), we concentrate on regionally identifying information in an effort to maximize geographic accuracy.

[^alpha_beta]: When georeferencing `ALPHA PRIMARY SCHOOL`, we would consider `ALPHA SECONDARY SCHOOL` to be a much better match than `BETA PRIMARY SCHOOL`.

Classifying information for each polling station-year is stored in election-specific `type` and `subtype` columns. These columns align temporally with the election-specific polling station names from which these data were extracted. For example, a polling station active in 2013 will have columns for its truncated name (`ALPHA`), its type (`SCH`), and its subtype (`PRI`); separate columns will hold analogous information for any prior elections during which this polling station was active.

### Cleaning strings
To improve matching further, we clean the strings of both our point (polling station) and polygon (administrative boundary) data. Most of the cleaning will likely be idiosyncratic to the locale of the data, like correcting for inconsistencies in transliteration, pronunciation, or abbreviation.

One technique we've found particularly helpful is to order the elements of each string. This corrects the particularly troublesome pattern strings sometimes following the pattern `SAMBURU EAST`, while the same constituency may be recorded in a second data set as `EAST SAMBURU`. The function `order_substr()`, defined in `code/functions.R`, splits each string of a character vector at any commas, whitespace, forward slashes, and hyphens; the output is a vector with those substrings ordered alphabetically and collapsed with a single space. Alternative regular expressions for splitting can be included in the `split` parameter; other strings may be used to separate the alphabetized substrings via the `collapse` parameter.[^order_substr]


```r
tuna <- c("tuna,skipjack", "tuna  , bluefin", "yellow-fin - tuna", "tuna,   albacore")
order_substr(tuna)
```

```
## [1] "skipjack tuna"   "bluefin tuna"    "fin tuna yellow" "albacore tuna"
```

```r
colors <- c("green/red", "yellow / blue", "orange purple")
order_substr(colors, collapse = "-", reverse = TRUE)
```

```
## [1] "red-green"     "yellow-blue"   "purple-orange"
```

[^order_substr]: Cleaning with the `order_substr()` function works poorly when substrings are inconsistently separated by the split pattern. For example, the vector of tuna species above hyphenates yellow-fin but leaves bluefin as a single word. Therefore, with the default split patterns, `order_substr()` splits "yellow-fin" prior to alphabetizing the string. (In this particular case, a good fix may be to adjust the regex to split at hyphens only when they are surrounded by whitespace.)

## Linking Polling Stations to Match Points
We define a function for each of several matching steps. These steps differ primarily in terms of which lower-level polygons they use (e.g., sublocation, old ward, new ward). To cut down on processing time, we run these functions on multiple cores via `mclapply()`.[^nb_parallel]

[^nb_parallel]: If you are running this code on a machine with less than 32GB of RAM, we suggest decreasing the `mc.cores` parameter for the matching processes.

Each matching function follows the same basic framework:

1. Block geographically by loading the match points and lower-level administrative units (e.g., sublocations) corresponding to a polling station's constituency.
2. Search for the lower-level administrative units belonging to the polling station.
3. Calculate the string distance between truncated match point names and the truncated polling station name.
4. Build spatial measures of match quality (e.g., minimum geographic distance from match points to the polling station's lower-level administrative units).
5. Subset to the match points with a string distance score less than a _manually_ determined threshold.
6. Check type and subtype alignment to use as another quality measure.
7. Among the matches with the minimum string distance score, pick those with the best quality scores.
8. Throw out matches that are too far from the polling station's lower-level administrative boundaries.

Below, we describe the blocking and fuzzy matching steps in detail before walking through the code for a simplified matching loop with a single polling station. Next, we use the example data sets to demonstrate the two functions we used first in our matching process, `match_lsl()` and `match_inf_loc13()`. Together, these functions georeference roughly 75 percent of Kenya's 1997-2013 polling stations. We discuss two techniques particular to these functions; their full definitions can be found in `code/functions.R`.

### Blocking with polygon boundaries
Our full data set of match points contains over 125,000 coordinates. Running matching algorithms for each of the nearly 26,000 polling stations would therefore be hugely expensive. To save time and RAM, we adapt a common record linkage technique, blocking, to a spatial context: we split our matching data geographically and use constituency code identifiers as a blocking key.

We first buffer our constituency polygons by 500 meters to account for error in the spatial data. Next, we build a list in which each element is the subset of match points contained by each of the buffered constituencies. We use a convenience function defined in `code/functions.R`, to save each object in a new directory: `data/mp_constb/`. By default, the name of each file matches the name of the list element to which the file corresponds. We repeat this process to split the sublocation polygons into constituency blocks.



```r
# Buffer old constituencies
l_const <- split(const_old, const_old$id)
l_constb <- mclapply(l_const, gBuffer, mc.cores = 6)

# Split match points by buffered constituency
mp_constb <- mclapply(l_constb, function(x) mp[x, ],
                      mc.cores = 6)
save_list(mp_constb, "data/mp_constb")

# Split sublocations by buffered constituency
sl_constb <- mclapply(l_constb, function(x) sl[x, ],
                      mc.cores = 6)
save_list(sl_constb, "data/sl_constb")
```

Each match points and sublocations subset is named for the old (i.e., pre-2013) constituency code in which the spatial features are contained. We use these constituency codes as blocking keys by searching for matches only within subsets corresponding to a polling station's constituency. For example, when searching for matches to a polling station assigned to constituency `067` in 2002, we load only the objects in `data/mp_constb/067.rds` and `data/sl_constb/067.rds`.

### Fuzzy string matching
We use the `stringdist` package to calculate [Jaro-Winkler distances](https://en.wikipedia.org/wiki/Jaro%E2%80%93Winkler_distance) between the polling stations data and the match points and polygon name strings [@stringdist]. The `stringdist` package supports a number of metrics for measuring string similarity. We've found the Jaro-Winkler distance to work best for our purposes. Results of `stringdist(a, b, method = "jw")` are scaled between 0 and 1, for which lower numbers reflect more similar strings.

Aside from supporting multiple string metrics, one advantage of `stringdist()` over `agrep()` is that it provides a sense of how the distance calculations change across comparisons. Inspecting the results by hand allows you to pick precise thresholds for the minimum acceptable string distance.


```r
set.seed(10)
sl_samp <- sample(ps$sl02[ps$sl02 != ""], 20)
sd_l <- lapply(sl_samp, stringdist,
               b = sl$SUBLOCATIO, method = "jw")

# Find sublocation strings with the minimum string distance score
# In this example, we'll just take the first match
sl_match <- sapply(sd_l, function(x) sl[x == min(x), ]$SUBLOCATIO[1])
sl_dist <- sapply(sd_l, min)

df <- data.frame(sl_samp, sl_match, sl_dist,
                 stringsAsFactors = FALSE)
df2 <- arrange(df, desc(sl_dist))

head(df2, 10)
```

```
##          sl_samp      sl_match    sl_dist
## 1  GACHEGETHIURI       GACHEGE 0.15384615
## 2         MUVUTI        MUPUTI 0.11111111
## 3        MARIGUT       MAREGUT 0.09523810
## 4    NAROK NGARE   NAROK NKARE 0.06060606
## 5       KALIBENE     KALIMBENE 0.03703704
## 6   NDARA SAGALA NDARA SAGALLA 0.02564103
## 7       MOGOBICH      MOGOBICH 0.00000000
## 8       KAPTIONY      KAPTIONY 0.00000000
## 9          A BAR         A BAR 0.00000000
## 10       KAKEANI       KAKEANI 0.00000000
```

### Walkthrough with one polling station

### Applying functions to example data

# Evaluating Point Matches

## E pluribus unum


# Imputing points

# Conclusion
+ Improvements: ML applications (random forests?)

# References