Using spatial data to understand the world
================

Elements of human behaviour can be understood (at least in part) as a
function of geography. This includes economics (which often have
distinct spatial patterns), election outcomes (often decided by discrete
geographic contests), crime and public health.

In this project we shall use spatial data to understand important social
phenomena, including why some parts of the United States suffer from
higher mortality rates from drugs and alcohol than others, and what
regional factors predict election outcomes.

# Studying spatial data

Spatial analysis is the application of statistical analysis to data with
a geographical aspect. We are going to start by examining the
relationship between two variables: poverty and death caused by drugs
and alcohol. The spatial aspect of this is that we are going to use
county-level data sourced from the *US Census Bureau* and the *Centers
for Disease Control and Prevention (CDC)*.

We are looking at loading, merging and comparing spatial data from
different sources, and then studying associations between them. We will
look at adding a third variable and fitting a regression to these dates,
and then creating interactive maps to examine the spatial patterns in
our data.

### The data

First let us upload the poverty dataset. This was obtained from **[the
United States Census
Bureau](https://factfinder.census.gov/faces/nav/jsf/pages/download_center.xhtml#none)**.

We want the file *poverty data - ACS\_16\_5YR\_S1702\_with\_ann.csv* in
the folder *poverty*. We can upload this using the `read.csv()` function
with the code:

``` r
poverty.data <- read.csv(
  "US/poverty/poverty data - ACS_16_5YR_S1702_with_ann.csv", 
    skip = 1, header = T)
```

Then we shall load the file *Drugs and alcohol Cause of Death,
1999-2016.csv* from the *mortality* folder. These data were obtained
from **[the Center for Disease Control
(CDC)](https://wonder.cdc.gov/controller/datarequest/D77;jsessionid=319C4597424371058DEED861A0559485)**
As this is a .txt file, we use the `read.table()` function to do this:

``` r
drugs.mort.data <- read.table(
  "US/mortality/Drugs and alcohol Cause of Death, 1999-2016.txt", 
    sep = '\t', header = T)
```

This dataset contains 9 variables and 2934 rows. Both datasets contain
the same number of rows, which in both cases are the number of countries
in the United States.

We want to join these datasets, or elements of them. However, the
variables we would use to do this, the code used to identify each
county, is currently labelled differently in each data frame:

``` r
names(poverty.data)[3]
```

    ## [1] "Geography"

``` r
names(drugs.mort.data)[2]
```

    ## [1] "County"

When two datasets have a variable named and structured identically, it
is simple to combine them. However, in this instance, the names of our
ID variables don???t match, even though their structure does:

``` r
head(poverty.data$Id2)
```

    ## [1] 1001 1003 1005 1007 1009 1011

``` r
head(drugs.mort.data$County.Code)
```

    ## [1] 1001 1003 1005 1007 1009 1011

Additionally, we need to recode at least one other variable name, and
since the `poverty.data` file contains so many variables, we do not want
to merge everything. Rather, we want to extract and merge certain
variables. We could do this in multiple steps. However, it is faster and
neater to do this with a single block of code. The following syntax can
be used to rename the ID variable in one of our datasets, and then
extract those variables we want from each file, before merging these
data into a single new data frame we call `country.data`:

``` r
library(dplyr)
library(plyr)

patterns <- c(" County| Parish| city")

county.data <- merge(
  poverty.data %>% 
    dplyr::rename(below.poverty.rate = 
             All.families....Percent.below.poverty.level..Estimate..Families)  %>% 
    dplyr::select(Id2, below.poverty.rate),
  drugs.mort.data %>% 
    dplyr::rename(Id2 = County.Code,
                mortality.rate = Crude.Rate) %>% 
    mutate(county = gsub(patterns, "", County))  %>%
    dplyr::select(Id2, county, mortality.rate),
  by="Id2")
```

These commands are derived from the `dplyr` package. Within the
`merge()` function we use piping to include nested commands to rename
and recode variable names using the `rename()` and `mutate()` functions
respectively, and to extract only those variables we wish to include in
the new merged frame with `select()`. As we discussed in previous weeks,
the pipe operator allows for a block of code consisting of a number of
functions to be chained together ??? the output of one function inserted
it into the next ??? from left to right. We also remove the words
???County???, ???Parish??? and ???city??? from county names, as these are not
replicated in the shapefiles we will use later to map these data.

We now have a new dataset with 4 variables and 2928 rows. Viewing the
first six rows of our new data frame, we can see the structure of these
data:

    ##    Id2 below.poverty.rate      county mortality.rate
    ## 1 1001                9.4 Autauga, AL           12.5
    ## 2 1003                9.3 Baldwin, AL           22.6
    ## 3 1005               20.0 Barbour, AL            9.0
    ## 4 1007               11.7    Bibb, AL           14.1
    ## 5 1009               12.2  Blount, AL           18.1
    ## 6 1011               25.3 Bullock, AL           11.1

The first row is the ID variable we used to merge our data frames. The
second, the percentage of each county???s families with incomes below the
poverty rate. Third, the names of each country (and the state within
which they sit). Finally, the rate of mortailities related by drugs and
alcohol (per 100,000 people).

There is one last edit to make to these data before we examine them. If
you run the code `levels(as.factor(county.data$mortality.rate))` you
will see that some of the observations for the variable `mortality.rate`
are coded as ???Unreliable???. This creates two issues for us. Those
observations are effectively missing for our purposes, and this converts
the variable to a string when we want it to be a number.

``` r
county.data <- county.data %>% 
  mutate(mortality.rate = ifelse(mortality.rate == 'Unreliable', 
                                 NA, as.character(mortality.rate)) %>%
           as.numeric(as.character(mortality.rate)))
```

Within the `mutate()` function we use a combination of a conditional
statement, which recode those values of `mortality.rate` that equal
???Unreliable??? as missing while retaining the value of all other
observations. It then converts the variable into a numeric vector.

We are now ready to study the association between these two variables at
the county level.

### Examining the association between county-level poverty and deaths associated with drug and alcohol use

We can use these data now to plot the association between the percentage
of families with incomes below the poverty rate. We do this by loading
the `ggplot2` and `scales` packages in *R*, and then use similar syntax
as we have used in the past to create a plot of our data, which I have
included below.

There are a few small differences to the previous code we have used. The
most significant is that we use `geom_jitter` to specify the use of
points in our graph, rather than `geom_point`. The `jitter` function is
a shortcut for `geom_point(position = "jitter")`. It adds random noise
to the location of each point in your plot, and is a useful way of
handling overplotting caused by discreteness in datasets (which is
something we experience with these data, which you can see by using
`geom_point`). We specify the amount of noise to add to these data with
`width` and `height`.

``` r
library(ggplot2)
library(scales)


ggplot(county.data, 
       aes(y=mortality.rate, x=(below.poverty.rate / 100))) +
  geom_jitter(size = .5, alpha = .25, width = .01, height = .5) +
  stat_smooth(se=FALSE, fullrange=TRUE) +
  labs(title = "County-level drug/alcohol related mortality rate\nversus poverty rate", 
       y = "Drug/alcohol related mortality rate\n(per 100,000 population)\n",  
       x = "Families below poverty level (%)") +
  scale_x_continuous(percent_format(accuracy = 1)) +
  theme_minimal() +
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.major.y = element_line(colour="grey", size=0.4), 
        panel.grid.minor = element_blank(), 
        legend.position="none", 
      plot.title = element_text(size=12, face="bold", vjust=2.7, hjust=.5), 
        strip.text.x = element_text(size=13, vjust=1), 
        strip.background = element_blank(), 
        axis.title.x = element_text(size=13, vjust=-.75), 
        axis.title.y = element_text(size=13, vjust=1.5), 
        axis.text.x = element_text(size=9, vjust=-.25), 
        axis.text.y = element_text(size=9, hjust=1))
```

Which produces this graph:

<div class="figure" style="text-align: center">

<img src="Using-spatial-data-to-understand-the-world_files/figure-gfm/mortality_poverty-1.png" alt="The association between county-level poverty rates and Drug/alcohol related mortality rate.\label{fig:mortality_poverty}"  />
<p class="caption">
The association between county-level poverty rates and Drug/alcohol
related mortality rate.
</p>

</div>

This suggests that there is a small but real association between the per
cent of families with incomes below the poverty line and the drug and
alcohol related mortality rate. We can understand this better by adding
some complexity into our analysis. Let us start by adding in some
information we already possess: state variation.

We currently do not have a variable for state in our data frame.
However, we do have information on the state in which a county is
located in within the `county` variable (which is structured
`county name`, `state abbreviation`). We can extract the state with the
`sapply()` and `strsplit` functions, using this code:

``` r
county.data$state <- sapply(strsplit(as.character(county.data$county) , "\\, ") , "[" , 2)
```

This tells *R* that we wish to split the string vector `county` in the
`county.data` frame (which we specify should be treated as a
???character???, otherwise this will not work) and then extract the second
of the new vectors, which we save as a new variable called `state`. We
can view the structure of this new variable with the `head()` and
`tail()` functions:

    ## [1] "AL" "AL" "AL" "AL" "AL" "AL"

    ## [1] "WY" "WY" "WY" "WY" "WY" "WY"

Before we plot these data with state included, we want to specify the
order in which the states will be graphed by `ggplot()`, which as you
might remember uses alphabetical order unless instructed otherwise.

With the following syntax we can calculate the correlation between
mortality rate and poverty rate by state, and then use this to reorder
the data frame by states:

``` r
county.data2 <- county.data %>% 
left_join(county.data %>% 
  group_by(state) %>% 
  dplyr::summarise(n = length(mortality.rate),
                   correlation = cor(mortality.rate, below.poverty.rate, 
                                            use='pairwise')) %>% 
    filter(n > 5)) %>% 
  na.omit() %>% 
  mutate(state = reorder(state, correlation)) %>% 
  arrange(state)
# county.data <- county.data[order(county.data$state),] 
```

Trying to ascertain the population-level relationship between
drug-related mortality and poverty in states with five or fewer counties
is likely pointless. We use the `length()` function to calculate the
number of counties and filter the dataset to only include states with
more than five counties.

We can then use the `facet_wrap()` command to facet our graph by state
(in the plot below I have also reverted to using `geom_point()` as
jittering is no longer required:

``` r
ggplot(county.data2, 
       aes(y=mortality.rate, x=(below.poverty.rate / 100))) +
  #geom_jitter(size = .5, alpha = .25, width = .01, height = .5) +
  geom_point(size = .5, alpha = .25) +
  stat_smooth(method = 'lm', se=FALSE, fullrange=TRUE) +
  labs(title = "County-level drug/alcohol related mortality rate versus poverty rate\nby state", 
       y = "Drug/alcohol related mortality rate\n(per 100,000 population)\n",  
       x = "Families below poverty level (%)") +
  scale_x_continuous(labels = percent_format(accuracy = 1)) +
  theme_minimal() +
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.major.y = element_line(colour="grey", size=0.4), 
        panel.grid.minor = element_blank(), 
        legend.position="none", 
      plot.title = element_text(size=12, face="bold", vjust=2.7, hjust=.5), 
        strip.text.x = element_text(size=13, vjust=1), 
        strip.background = element_blank(), 
        axis.title.x = element_text(size=13, vjust=-.75), 
        axis.title.y = element_text(size=13, vjust=1.5), 
        axis.text.x = element_text(size=9, vjust=-.25), 
        axis.text.y = element_text(size=9, hjust=1)) + 
  facet_wrap( ~ state, ncol = 5, scales = 'free')
```

![The association between county-level poverty rates and Drug/alcohol
related mortality rate, by
state.](Using-spatial-data-to-understand-the-world_files/figure-gfm/mortality_poverty_state-1.png)

Note `scales = 'free'` within the `facet_wrap()` function to allow the
scales and breaks of the axes to vary by the data within each facet
plot.

This figure adds to our analysis by showing that there is state-level
variation in the relationship between poverty and mortality associated
with drugs and alcohol. We can add further to this study though by
including more data and fitting a regression to these data.

### Running a regression with these data

We also have information on the number of prescriptions issued in a
given county. This can be found in the *opioids* folder, saved as the
file *opioids prescribing rates*. We can upload this using the
`read.csv()` function with the code:

``` r
opioids.data <- read.csv("US/opioids/opioids prescribing rates.csv")
```

We then use the same steps as above to merge this data with our
`county.data` frame:

``` r
county.data <- merge(
  county.data  %>%
  dplyr::mutate(z.poverty = scale(below.poverty.rate)), 
  opioids.data %>%
    dplyr::mutate(county = ??..County) %>%
  dplyr::mutate(z.opiates = scale(X2016.Prescribing.Rate)) %>%
    dplyr::select(county, z.opiates, X2016.Prescribing.Rate),
  by = "county")
```

In this code, we also standardise the poverty presceription rates in the
same line of block of code that we use to merge the data.

We are then ready to fit our first model. Specify this as a generalised
linear model (glm) by using the `glm()` function. We want to use this to
fit a Poisson regression. These are glms using the logarithm as the link
function (instead of the logit in logistic regression), and the Poisson
distribution function as the assumed probability distribution of the
response (instead of the binimomial as we do in logistic). These models
are useful when predicting an outcome variable representing counts from
a set of continuous predictor variables. In this case, we are looking at
the mortality rate in counties per 100,000 people; which is a count of
deaths as a proportion of the population. As it has a hard floor of zero
(the mortality rate cannot be negative) Poisson is appropriate here.

``` r
mortality.model.1 <- glm(mortality.rate ~ z.poverty + z.opiates, 
                         family=poisson(link = log),
                         data=county.data)
```

Load the `arm` package and then use the `display()` function to view the
results of your model:

``` r
library(arm)

display(mortality.model.1)
```

    ## glm(formula = mortality.rate ~ z.poverty + z.opiates, family = poisson(link = log), 
    ##     data = county.data)
    ##             coef.est coef.se
    ## (Intercept) 3.01     0.00   
    ## z.poverty   0.08     0.00   
    ## z.opiates   0.08     0.00   
    ## ---
    ##   n = 2594, k = 3
    ##   residual deviance = 8628.5, null deviance = 9538.3 (difference = 909.8)

This indicates both the poverty rate and the availability of opiates
have a relationship with deaths due to drug or aclcohol use.

We can understand this further by fitting a multilevel linear regression
model to these data. This model allows us to vary the intercept in our
regression by state, taking the state-level variation into account.

``` r
mortality.model.2 <- glmer(mortality.rate ~ z.poverty + z.opiates + (1 | state), 
                         family=poisson(link = log),
                         data=county.data)
```

`lmer()` specifies that this is a multilevel regression model, and (1 \|
state) that we want to allow intercepts to vary by state (with no
varying slopes). The resuls from this model are:

``` r
display(mortality.model.2)
```

    ## glmer(formula = mortality.rate ~ z.poverty + z.opiates + (1 | 
    ##     state), data = county.data, family = poisson(link = log))
    ##             coef.est coef.se
    ## (Intercept) 3.09     0.14   
    ## z.poverty   0.10     0.01   
    ## z.opiates   0.11     0.00   
    ## 
    ## Error terms:
    ##  Groups   Name        Std.Dev.
    ##  state    (Intercept) 1.00    
    ##  Residual             1.00    
    ## ---
    ## number of obs: 2594, groups: state, 50
    ## AIC = Inf, DIC = -Inf
    ## deviance = 4898.0

Which are similar to the classical linear model. You can view the
varying intercepts with the `ranef()` function, and the standard errors
with `se.ranef()`. The code for this (linked with a column bind) is:

``` r
cbind(ranef(mortality.model.2)$state, se.ranef(mortality.model.2)$state)
```

With the first six rows of the printout being:

``` r
head(cbind(ranef(mortality.model.2)$state, se.ranef(mortality.model.2)$state))
```

Suggesting the state-level effects are mostly small and noisy, with the
only exceptions from the first six states being Rhode Island and South
Carolina.

We can examine how this works by, for instance, comparing estimates of
the mortality rate of countries in California and new Mexico with high
and low levels of opiate use, and an average poverty rate. We can do
this with the code:

``` r
#California 

exp(coef(mortality.model.2)$state['CA','(Intercept)'] + #intercept for California
  coef(mortality.model.2)$state['CA','z.opiates'] * c(-1, 1)) #coefficient for opiate use
```

    ## [1] 26.38260 32.61377

``` r
#West Virginia

exp(coef(mortality.model.2)$state['NM','(Intercept)'] + #intercept for New Mexico
  coef(mortality.model.2)$state['NM','z.opiates'] * c(-1, 1)) #coefficient for opiate use
```

    ## [1] 37.27305 46.07638

This pulls the intercept for California and New Mexico, respectively,
and sums this with the coefficient for opiod use multiplied by negative
one and one (providing us with the coefficient values for one standard
error below and above the mean). We then exponentiate this, to reverse
the log transformation performed by our Poisson regression. This is the
equivalent of using the inverse logit function for logistic regression.

### Creating interactive maps in *R*

Now time for the fun part. We are ready to move on to makes some
interactive maps with these data.  
To do this, we need to load shapefiles. These provide the information
for the shapes on the map for the geographic units we are plotting data
for. They do so using polygons, which we will overlay on background map
tiles. In this instance, we are working with a shapefile containing data
on the boundaries of American counties, which matches our data.

``` r
library(sf)

us.shape.1 <- st_read("US/US shapefiles/cb_2016_us_county_20m.shp")
```

    ## Reading layer `cb_2016_us_county_20m' from data source 
    ##   `D:\Document\DS Work\My Github Repositories\Spatial-data-to-understand-the-world\US\US shapefiles\cb_2016_us_county_20m.shp' 
    ##   using driver `ESRI Shapefile'
    ## Simple feature collection with 3220 features and 9 fields
    ## Geometry type: MULTIPOLYGON
    ## Dimension:     XY
    ## Bounding box:  xmin: -179.1743 ymin: 17.91377 xmax: 179.7739 ymax: 71.35256
    ## Geodetic CRS:  NAD83

We then want to link these shapefiles to the county results. We start by
adding the state name to the shapefile data. To do this we load a
dataset saved in the US geospatial data folder called *fips
concordance.csv*:

``` r
fip.concordance <- read.csv("US/fips concordance.csv")
```

This file, which we were required to modify a little above, can be
matched to the shapefile data using similar syntax to above:

``` r
us.shape.2 <- us.shape.1 %>% 
  merge(fip.concordance %>%
          mutate(STATEFP =  ifelse(Numeric.code >= 0 & Numeric.code <= 9, 
                             paste0(0 , Numeric.code), Numeric.code)) %>%
          rename(replace=c("Alpha.code" = "state")),
        by="STATEFP")
```

We edit the variable containing county names in shapefile using the
`paste0()` function, which combines the county name with the state
abbreviation (from the fip concordance file). We then modify county
names with some different spelling in the shapefiles and our county
level data:

``` r
us.shape.2$county <- paste0(us.shape.2$NAME, ", ", us.shape.2$state)

us.shape.2$county <- gsub("Do??a Ana, NM", "Dona Ana, NM", us.shape.2$county)
us.shape.2$county <- gsub("LaSalle, LA", "La Salle, LA", us.shape.2$county)
us.shape.2$county <- gsub("Oglala Lakota, SD", "Oglala, SD", us.shape.2$county)

us.shape.2$county[is.na(us.shape.2$below.poverty.rate)]  
```

    ## character(0)

And then we merge the shapefile and the county-level data:

``` r
us.shape.2 <- merge(us.shape.2,
  county.data %>%
    dplyr::select(county, below.poverty.rate, mortality.rate, X2016.Prescribing.Rate),
  by="county", duplicateGeoms = TRUE)
```

We are now ready to map this. To do this we are using the `mapdeck`
package. Install and then load this now.

Download some other packages from *Github*.

``` r
#install.packages("devtools")
library(devtools)

devtools::install_github("SymbolixAU/jsonify", force = TRUE)
devtools::install_github("dcooley/sfheaders", force = TRUE)
devtools::install_github("SymbolixAU/geojsonsf", force = TRUE)
devtools::install_github("SymbolixAU/colourvalues", force = TRUE)
devtools::install_github("SymbolixAU/spatialwidget", force = TRUE)
devtools::install_github("SymbolixAU/mapdeck")
```

Mapbox account creation and organise token from
[here](https://account.mapbox.com/auth/signin/?route-to=%22/access-tokens/%22).

Then store your token in *R* as an item named `key` with the code:

``` r
key <- 'pk.eyJ1IjoianViYWllciIsImEiOiJjazQzaDJ3OHIwN3Q4M2txeTVpcmxwbHFpIn0.Lzhd9uoNIkWq6UyobiJGYQ'
```

Once this is done, map of the poverty rate by county.

To create our interactive maps, we call the `mapdeck()` function. Within
this, we enter our token; which allows us to call data through it. we
then specify the style of our maps with `mapdeck_style()`. In this case,
we use the ???light??? style; which has minimal background information (and
therefore distractions).

We use a dplyr pipe to connect our `mapdeck` call with the
`add_polygon()` function, within which we specify and populate our
polygons. Here we specify the data we are using comes from the
shapefiles we stored in `us.shape.2`. We specify we want this to appear
as a polygon layer, and that we wish to colour the fill of our polygons
using the variable `below.poverty.rate`. Doing this will shade US
counties with high poverty rates a different colour than those with low
rates of poverty. Finally, we also specify the colour palette we wish to
use to fill the polygons (???diverge\_hsv???), the opacity of the polygons
(0 is completely translucent, 1 completely opaque), and we tell
`mapdeck` we wish to include a legend.

``` r
mapdeck(token = key, style = mapdeck_style("light")) %>%
  add_polygon(
    data = us.shape.2, 
    layer = "polygon_layer", 
    fill_colour = "below.poverty.rate", 
    palette = "diverge_hsv", 
    fill_opacity = .9, 
    legend = TRUE
  )  
```

When execute this code, map will appear in the markdown file below the
code that created it.

We can replicate this with the mortality data:

``` r
library(tidyverse)

us.shape.mortality <- us.shape.2 %>% 
  drop_na(mortality.rate)


mapdeck(token = key, style = mapdeck_style("light")) %>%
  add_polygon(
    data = us.shape.mortality, 
    layer = "polygon_layer", 
    fill_colour = "mortality.rate", 
    palette = "diverge_hsv", 
    fill_opacity = .9, 
    legend = TRUE
  )  
```

When doing this we have stored the shapefile data into a different item
after using the `drop_na()` function to remove those counties from the
dataset for which we have `NA` values for the variable `mortality.rate`.
We do this as these will now just be empty spaces. If we had not, they
would be shaded grey; which is more distracting and takes our attention
away from the counties we should be focusing on (those for which we have
data).

We can also do this for the rates that opiates are prescribed in
different counties.

``` r
# aust.2$e <- aust.2$density

us.shape.prescription <- us.shape.2 %>% 
  drop_na(X2016.Prescribing.Rate)


mapdeck(token = key, style = mapdeck_style("light")) %>%
  add_polygon(
    data = us.shape.prescription, 
    layer = "polygon_layer", 
    fill_colour = "X2016.Prescribing.Rate", 
    palette = "diverge_hsv", 
    fill_opacity = .9, 
    legend = TRUE
  )  
```
