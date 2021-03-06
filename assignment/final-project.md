California ISO Electric Grid Data Analysis
================
Michaela Palmer & Melissa Ferriter

``` r
knitr::opts_chunk$set(echo = T, warning = F, message = F)
```

``` r
libs <- c("tidyverse", "foreign", "zoo", "ggpubr", "forecast")
sapply(libs, require, character.only=T)
```

``` r
theme_set(theme_minimal()) # Set theme to theme_minimal as base
# Edit some of the parameters in the minimal theme
theme_update(plot.title = element_text(hjust=0.5))
```

Data Preparation and Reading
----------------------------

The data used in this analysis comes from [California Independent System Operator Corporation (CAISO)](http://www.caiso.com/green/renewableswatch.html), which aggregates grid data from electricity producers in California.

``` r
# Create and view an object with file names & full paths
days <- c(paste0("0", 1:9), 10:25)
months <- c(paste0("0", 1:9), 10:12)

urls <- list()
for (i in months) {
  url <- paste0("http://content.caiso.com/green/renewrpt/2017", i, days,"_DailyRenewablesWatch.txt")
  urls[[paste0("month", i)]] <- url
}

# Decemeber 2017 still in progress, can only partially import 
dec_days <- c(paste0("0", 1:9), 10:11)
dec_url <- paste0("http://content.caiso.com/green/renewrpt/2017", 12, dec_days,"_DailyRenewablesWatch.txt")

# Import December 2016 for seasonal analysis 
dec2016 <- paste0("http://content.caiso.com/green/renewrpt/2016", 12, days,"_DailyRenewablesWatch.txt")

# Import March 2015 and 2016 for predictions
mar15 <- paste0("http://content.caiso.com/green/renewrpt/2015", "03", days,"_DailyRenewablesWatch.txt")
mar16 <- paste0("http://content.caiso.com/green/renewrpt/2016", "03", days,"_DailyRenewablesWatch.txt")
```

We are going to be reading in the daily CAISO data for 2017 and aggregating it by month into individual dataframes. However, we have to be aware that this is unverified raw data that contains errors. Consequently, we chose to not import March 2017 due to the warning:

`"The supplied DateTime represents an invalid time. For example, when the clock is adjusted forward, any time in the period that is skipped is invalid."`

``` r
# Function to import data with proper formatting & columns 
import <- function(data) {
  data.frame(date = as.Date(basename(data), "%Y%m%d"), readr::read_table2( 
    data,
    col_names = c("hour", "geothermal", "biomass", "biogas", "small_hydro", "wind", "solar_pv", "solar_thermal" ),
    skip = 2,
    n_max = 24
  )) 
}

jan <- do.call("rbind", lapply(urls[["month01"]], import))
feb <- do.call("rbind", lapply(urls[["month02"]], import))
march15 <- do.call("rbind", lapply(mar15, import))
march16 <- do.call("rbind", lapply(mar16, import))
april <- do.call("rbind", lapply(urls[["month04"]], import)) 
may <- do.call("rbind", lapply(urls[["month05"]], import)) 
june <- do.call("rbind", lapply(urls[["month06"]], import))
july <- do.call("rbind", lapply(urls[["month07"]], import))
aug <- do.call("rbind", lapply(urls[["month08"]], import))
sept <- do.call("rbind", lapply(urls[["month09"]], import))
oct <- do.call("rbind", lapply(urls[["month10"]], import))
nov <- do.call("rbind", lapply(urls[["month11"]], import))
dec <- do.call("rbind", lapply(dec_url, import))
dec16 <- do.call("rbind", lapply(dec2016, import))
```

When reading in the dates from the urls, we tested out other regular expressions methods to access the year, month, a day from the `.txt` file. These regex methods below work, but we decided to pursue the `as.Date` function in base R for sake of conciseness.

``` r
sub(".*(\\d{8}).*", "\\1", urls[["month01"]])
```

    ##  [1] "20170101" "20170102" "20170103" "20170104" "20170105" "20170106"
    ##  [7] "20170107" "20170108" "20170109" "20170110" "20170111" "20170112"
    ## [13] "20170113" "20170114" "20170115" "20170116" "20170117" "20170118"
    ## [19] "20170119" "20170120" "20170121" "20170122" "20170123" "20170124"
    ## [25] "20170125"

``` r
gsub("(?:.*/){4}([^_]+)_.*", "\\1", urls[["month01"]])
```

    ##  [1] "20170101" "20170102" "20170103" "20170104" "20170105" "20170106"
    ##  [7] "20170107" "20170108" "20170109" "20170110" "20170111" "20170112"
    ## [13] "20170113" "20170114" "20170115" "20170116" "20170117" "20170118"
    ## [19] "20170119" "20170120" "20170121" "20170122" "20170123" "20170124"
    ## [25] "20170125"

``` r
gsub("\\D", "", urls[["month01"]])
```

    ##  [1] "20170101" "20170102" "20170103" "20170104" "20170105" "20170106"
    ##  [7] "20170107" "20170108" "20170109" "20170110" "20170111" "20170112"
    ## [13] "20170113" "20170114" "20170115" "20170116" "20170117" "20170118"
    ## [19] "20170119" "20170120" "20170121" "20170122" "20170123" "20170124"
    ## [25] "20170125"

``` r
sub(".*/", "", sub("_.*", "", urls[["month01"]]))
```

    ##  [1] "20170101" "20170102" "20170103" "20170104" "20170105" "20170106"
    ##  [7] "20170107" "20170108" "20170109" "20170110" "20170111" "20170112"
    ## [13] "20170113" "20170114" "20170115" "20170116" "20170117" "20170118"
    ## [19] "20170119" "20170120" "20170121" "20170122" "20170123" "20170124"
    ## [25] "20170125"

``` r
sub(".*(\\d{8}).*", "\\1", urls[["month01"]])
```

    ##  [1] "20170101" "20170102" "20170103" "20170104" "20170105" "20170106"
    ##  [7] "20170107" "20170108" "20170109" "20170110" "20170111" "20170112"
    ## [13] "20170113" "20170114" "20170115" "20170116" "20170117" "20170118"
    ## [19] "20170119" "20170120" "20170121" "20170122" "20170123" "20170124"
    ## [25] "20170125"

To solve for the missing data issue in March 2017, we tried to forecast values based on 2015 and 2016 generation. Forecasts have been produced by applying `forecast()` directly to each time series. Thisselects an ETS model using the AIC, estimates the parameters, & generates forecasts. Although it returns prediction intervals, we extract the individual forecasts. (We are using mean in the returned forecast object because they are usually the mean of the forecast distribution.)

``` r
marchs <- dplyr::bind_rows(march15, march16)
marchs <- forecast::msts(marchs[,-c(1:2)], c(24, 7*24, 50*24)) # hourly seasonality
ns <- ncol(marchs)
h <- 600
march.fcast <- matrix(NA,nrow=h,ncol=ns,
                      dimnames= list(NULL, c("geothermal", "biomass", "biogas", "small_hydro", "wind", "solar_pv", "solar_thermal")))
for(i in 1:ns)
  march.fcast[,i] <- forecast::forecast(marchs[,i],h=h)$mean
```

Using the forecasted values, we can create a dataframe for March 2017. However, this model is not the most statistically robust for predicating generation in March, so we will not be including it in the analysis. Regardless, it was a very interesting exercise.

``` r
march <- data.frame(date = seq(as.Date("2017-03-01"), as.Date("2017-03-26"), length.out = 601)) %>%
  slice(1:600)
  march["hour"] <- c(1:24)

march.vals <- as.data.frame(march.fcast)  
march <- dplyr::bind_cols(march, march.vals)
march <- as.data.frame(march)
```

Preparing Processed Data for Archiving/Publication
--------------------------------------------------

We aggregate all of the scrapped/processed data into a single `csv.` file, intended to be used for publication or further processing.
This `csv` contains an hourly breakdown of total production by resource type given for Renewables (including Solar, Thermal, Solar, Wind, Small Hydro, Biogas, Biomass, and Geothermal), Nuclear, Thermal, Imports, and Hydro within the ISO grid from January to December 2017, with the exception of March. There is a column for each energy generation source, as well as a hour (24hrs) and date column (YYYY-MM-DD).

``` r
caiso17.full <- dplyr::bind_rows(jan, feb, april, may, june, july, aug, sept, oct, nov, dec)
write.csv(caiso17.full, file = "CAISO_2017FullRenewrpt.csv", row.names = F)
```

Daily Average Data Plots - By season
------------------------------------

The CAISO data are provided at hourly intervals. Plotting the data for each month the entire 6-year period would have generated an overwhelming number of graphs, so we first wrote a funcntion to calculate daily means and plot them.

``` r
plot <- function(data, name) {
  data[,-c(2)] %>%
    zoo::read.zoo(FUN = identity) %>%
    aggregate(as.Date, mean) %>%
    zoo::fortify.zoo() %>%
    dplyr::select(Index, geothermal, biomass, biogas, small_hydro, wind, solar_pv, solar_thermal) %>%
    tidyr::gather(Renewables, mean,-Index) %>%
    ggplot() +
    geom_line(mapping = aes(x = Index, y = mean, color = Renewables)) + 
    labs(title = paste(name, "2017 Daily Averages", sep = " "), x = "Date", y = "Generation (MW)") +
    theme(legend.position="bottom") 
}
```

Exploring High-demand/Peak Periods during August and September.
---------------------------------------------------------------

Demand/peak periods during August and September.

``` r
aug %>%
  dplyr::filter(hour >= 16, hour <= 21 ) %>%
  plot(name = "August: Peak Demand (4-9pm)")  
```

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-9-1.png)

``` r
plot(aug, name = "August 24hr")
```

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-9-2.png)

``` r
sept %>%
  filter(hour >= 16, hour <= 21 ) %>%
  plot(name = "September: Peak Demand (4-9pm)")  
```

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-9-3.png)

``` r
plot(sept, name = "September 24hr")
```

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-9-4.png)

Next, we are going to categorize the months under the four seasons. The data begins in December 2016 and extends to November 2017. Assumptions:

-   December - February -&gt; Winter
-   April - May -&gt; Spring
-   June - August -&gt; Summer
-   September - November -&gt; Fall

We aggregated the months by these seasonal assumptions and plotted them below.

``` r
winter <- bind_rows(dec16, jan, feb)
spring <- bind_rows(april, may)
summer <- bind_rows(june, july, aug)
fall <- bind_rows(sept, oct, nov)
season.list <- list("Winter 2016 -"=winter, Spring=spring, Summer=summer, Fall=fall)
Map(plot, data = season.list, name = names(season.list))
```

    ## $`Winter 2016 -`

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-10-1.png)

    ## 
    ## $Spring

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-10-2.png)

    ## 
    ## $Summer

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-10-3.png)

    ## 
    ## $Fall

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-10-4.png)

We can explore seasonality/time series analysis using these visualizations. WeWe are particulary interested in this analysis because summer irradiance &gt; winter irradiance, meaning that there should be greater generation in the summer months. The magnitude of generation is the least in Winter, to be expected. Another interesting feature of these plots is that that wind generation peaks around mid-year and decreases to low generation in the winter, similar to solar, but more pronounced.

Peak and Super Peak Generation by Source
----------------------------------------

Next, we are going to explore which sources have the most output during peak (Jan-Feb 4-9pm) and super peak (July-Aug 4-9pm) time frames. We expect that some sources will maintain constant output, while others, such as solar, will experience varied output depending on month and time of day.

``` r
# visualize how renewables change throughout the peak and super peak periods

pk <- bind_rows(jan, feb)
spk <- bind_rows(july, aug)

peak <- pk[, -c(1)] %>%
    filter(hour >= 16, hour <= 21 ) %>%
    group_by(hour) %>%
    dplyr::summarise_all(funs(mean)) %>%
    gather(Renewables, mean, -hour) %>%
    ggplot() +
    geom_line(mapping = aes(x = hour, y = mean, color = Renewables)) + 
    labs(title = "Peak Time Frame (Jan-Feb 4-9pm)", x = "Hour", y = "Generation (MW)") 

super.peak <- spk[, -c(1)] %>%
    filter(hour >= 16, hour <= 21 ) %>%
    group_by(hour) %>%
    summarise_all(funs(mean)) %>%
    gather(Renewables, mean, -hour) %>%
    ggplot() +
    geom_line(mapping = aes(x = hour, y = mean, color = Renewables)) + 
    labs(title = "Super Peak Time Frame (July-August 4-9pm)", x = "Hour", y = "Generation (MW)") 

ggpubr::ggarrange(peak + rremove("xlab") + rremove("x.text"), super.peak, nrow = 2, common.legend = T, legend = "bottom")
```

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-11-1.png)

As evidenced by the graphs, it would be prudent to utilize a mix of both solar and wind energy during peak and super peak periods.
We see that, as expected, solar is the most abundant source during daylight hours, and then wind becomes the dominant source at approximately 5pm and 7pm in winter and summer respectively. Despite its intermittent nature, solar outperforms any current renewable in generation potential.

``` r
# Compare the averages over the peak and super peak time frames for each renewable
peak.avg <- pk[, -c(1)] %>%
    filter(hour >= 16, hour <= 21 ) %>%
    select(everything(), -hour) %>%
    summarise_all(funs(mean)) %>%
    gather(Renewable, "Peak Mean")

super.peak.avg <- spk[, -c(1)] %>%
    filter(hour >= 16, hour <= 21 ) %>%
    select(everything(), -hour) %>%
    summarise_all(funs(mean)) %>%
    gather(Renewable, "Super Peak Mean")

final <- merge(peak.avg, super.peak.avg, by = "Renewable")
final
```

    ##       Renewable  Peak Mean Super Peak Mean
    ## 1        biogas  182.51333        172.7400
    ## 2       biomass  205.38667        253.9900
    ## 3    geothermal  947.87667        962.9433
    ## 4   small_hydro  416.95667        579.7133
    ## 5      solar_pv  717.37667       3983.2300
    ## 6 solar_thermal   26.92667        229.8933
    ## 7          wind 1261.70000       2490.7033

``` r
# Vizualization of peak and super peak averages by renewable source
final %>%
  gather(Time, Mean, -Renewable) %>%
  ggplot(aes(x = Renewable, y = Mean, fill = Time)) + geom_bar(stat="identity", position = "dodge", width = 0.6) +
  scale_fill_brewer(palette="Set1") + guides(fill=guide_legend("Period")) + 
  labs(title = "Peak vs Super Peak Renewable Generation",
    x = "Renewable Source", 
    y = "Generation (MW)") + 
    theme(legend.position = c(0.2, 0.8))
```

![](final-project_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-13-1.png)

This graph of renewable generation averaged over peak and super peak time periods is a way to visualize how resources vary throughout the year, both within and between themselves. It is evident that there is a large disparity between winter and summer output for solar due to light availability, while geothermal remains a consistent source year-round. Similarly, Small Hydro is fairly consistent, although it generates much less electricity.
