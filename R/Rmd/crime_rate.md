Crime data 2013-2015
================

Crime rates, annualized 2013-2015
---------------------------------

Input data isn't included on GitHub.

``` r
library(tidyverse)
library(stringr)
library(readxl)
library(tigris)
library(rgeos)
library(forcats)
library(lubridate)
library(acs)
```

``` r
crime <- read_excel("~/_R/ctp-dash/data/nhcrime13_15_nbhd_ct10_ucr.xlsx")
crime_df <- crime %>% select(NIBRS_Offe, Date_Occur, X, Y, STFID) %>%
  setNames(c("offense", "date", "long", "lat", "tract")) %>%
  mutate(offense = str_to_lower(offense))
```

``` r
offenses <- crime_df %>% count(offense) %>% arrange(desc(n))

crime_cat <- crime_df %>%
  mutate(cat = as.factor(offense) %>%
           fct_collapse("criminal homicide" = c("murder & non negligent manslaughter", "negligent manslaughter"),
          "forcible rape" = c("forcible rape"),
          "robbery" = c("robbery", "robbery with a firearm"),
          "aggravated assault" = c("aggravated assault", "assault with a firearm"),
          "burglary" = c("burglary-breaking & entering", "burglary attempt"),
          "larceny" = c("larceny-from building", "larceny-from vehicle", "larceny - all other", "larceny-shoplifting", "larceny-of m/v parts & accessories", "larceny-pocket-picking", "larceny-purse snatching", "larceny-from coin operated device", "larceny 6th deg"),
          "motor vehicle theft" = c("theft of auto", "motor vehicle theft (nh steal)"),
          "arson" = c("arson") # ,
          # "simple assault" = c("simple assault")
          ) %>% 
           fct_other(keep = c("criminal homicide", "forcible rape", "robbery", "aggravated assault", "burglary", "larceny", "motor vehicle theft", "arson")))

part1 <- crime_cat %>% 
  filter(cat != "Other") %>%
  mutate(type = fct_collapse(cat, "violent" = c("criminal homicide", "forcible rape", "robbery", "aggravated assault"), "property" = c("burglary", "larceny", "motor vehicle theft", "arson")))

# covers 3 years
range <- (interval(min(part1$date, na.rm = T), max(part1$date, na.rm = T)) / dyears(1)) %>% round(digits = 1)
```

``` r
pops <- acs.fetch(geography = geo.make(state = 09, county = 09, tract = "*"), endyear = 2015, table.number = "B01003", col.names = "pretty")

pop_df <- data.frame(tract = pops@geography$tract, pops@estimate) %>%
  tbl_df() %>%
  setNames(c("tract", "population"))
pop_df
```

    ## # A tibble: 190 x 2
    ##     tract population
    ##  * <fctr>      <dbl>
    ##  1 120100       6013
    ##  2 120200       6783
    ##  3 125100       4477
    ##  4 125200       5912
    ##  5 125300       5164
    ##  6 125400       3482
    ##  7 130101       4789
    ##  8 130102       2906
    ##  9 130200       8850
    ## 10 140100       3500
    ## # ... with 180 more rows

``` r
# get fips used in 500 cities to filter
fips <- read_csv("../input/500_cities.csv") %>%
  filter(GeographicLevel == "Census Tract") %>%
  select(TractFIPS) %>%
  unique() %>%
  mutate(fips = str_sub(TractFIPS, -6, -1)) %>%
  `[[`("fips")
```

``` r
part1_count <- part1 %>% 
  filter(!is.na(tract)) %>% 
  count(type, tract) %>%
  mutate(tract = str_sub(tract, -6, -1)) %>%
  filter(tract %in% fips) %>%
  left_join(pop_df, by = "tract") %>%
  mutate(rate1k = n / population * 1000) %>%
  mutate(rate1kyr = rate1k / range)
```

``` r
part1_count %>%
  mutate(name = paste0("09009", tract), indicator = "annual crime rate per 1000") %>%
  mutate(year = "2013-2015") %>%
  select(name, indicator, year, type, value = rate1kyr) %>%
  write_csv("../output/crime_rate_by_tract.csv")
```

``` r
part1_count %>%
  group_by(type) %>%
  summarise(total = sum(n), city_pop = sum(population)) %>%
  mutate(rate1k = total / city_pop * 1000) %>%
  mutate(rate1kyr = rate1k / range)
```

    ## # A tibble: 2 x 5
    ##       type total city_pop    rate1k  rate1kyr
    ##     <fctr> <int>    <dbl>     <dbl>     <dbl>
    ## 1  violent  3804   130612  29.12443  9.708143
    ## 2 property 17464   130612 133.70900 44.569667
