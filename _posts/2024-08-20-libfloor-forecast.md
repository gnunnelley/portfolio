---
layout: post 
title: "Time Series Analysis -- Library Floor"
date: 2024-08-20 01:00:00 -0700
output:
  md_document:
    variant: markdown_github
    preserve_yaml: TRUE
---

The following post details the application of an ARIMA model to forecast the occupancy of a library floor dataset with hourly increments. The code is written in R and is rendered from an rmarkdown. This is personal project combining a dataset from a prior project and methodology associated with a past course.

## Data Cleaning 

The data contains multiple floors of the library. I filtered the dataset down to the averages for the entirety of the library. The *average_occupancy* variable represents the average number of occupants hourly. The project focuses on a univariate analysis of the average occupants for the entire library. More information and exploratory analysis on the dataset can be found [here](https://dailynexus.com/2023-11-03/the-best-time-to-hit-the-books-exploring-occupancy-trends-in-the-ucsb-library/). There are gaps of time in the dataset, most of them occuring in September 2022. Later on, I use the ts_tibble to handle these gaps. 

``` r
floor = read_csv('libFloor.csv', show_col_types = FALSE)
floor.ts = floor[(floor$Location == 'UCSB Library'), c("Timestamp", "Average Occupancy", "Peak Occupancy")] #, "Quarter", "Hour")]
floor.ts <- floor.ts %>% clean_names()
floor.ts
```

    ## # A tibble: 14,386 × 3
    ##    timestamp           average_occupancy peak_occupancy
    ##    <dttm>                          <dbl>          <dbl>
    ##  1 2021-09-22 16:00:00                12              2
    ##  2 2021-09-25 12:00:00                30             39
    ##  3 2021-09-25 13:00:00                21             27
    ##  4 2021-09-25 14:00:00                91            199
    ##  5 2021-09-25 15:00:00                73             84
    ##  6 2021-09-25 16:00:00                56             64
    ##  7 2021-09-25 17:00:00                44             60
    ##  8 2021-09-25 18:00:00                13             34
    ##  9 2021-09-25 19:00:00                 3              5
    ## 10 2021-09-25 20:00:00                 1              3
    ## # ℹ 14,376 more rows


## Diagnostics and Transformation

### KSPSS and ADF

The series is more statistically significant when differenced once
versus not. The univariate time series of is observably
not stationary. The data is non-stationary and needs to be differenced
once.

``` r
occ_ts <- ts(floor.ts$average_occupancy)

occ_ts %>%
  diff() %>% 
  ur.kpss() %>%
  summary()
```

    ## 
    ## ####################### 
    ## # KPSS Unit Root Test # 
    ## ####################### 
    ## 
    ## Test is of type: mu with 13 lags. 
    ## 
    ## Value of test-statistic is: 7e-04 
    ## 
    ## Critical value for a significance level of: 
    ##                 10pct  5pct 2.5pct  1pct
    ## critical values 0.347 0.463  0.574 0.739

``` r
Box.test(occ_ts, type = "Ljung-Box") # 24 lags for hourly data
```

    ## 
    ##  Box-Ljung test
    ## 
    ## data:  occ_ts
    ## X-squared = 13316, df = 1, p-value < 2.2e-16

### Box-Cox

[Forecasting within limits](https://robjhyndman.com/hyndsight/forecasting-within-limits/)

Because the data often drops to near zero numbers a [lambda transformation](https://otexts.com/fpp2/transformations.html) ensures that the forecast is non-negative.

    ## Warning in guerrero(x, lower, upper): Guerrero's method for selecting a Box-Cox
    ## parameter (lambda) is given for strictly positive data.

    ## [1] 0.3083226

Holidays and weeks of the quarter impact the frequency of the
occupancy rates. More students attend the library the week before
finals and less students attend the library the week around
thanksgiving. The repetitions over school year makes me believe that
some seasonal pattern exists. The hourly frequency of the time
series makes the seasonality 
[complex](https://otexts.com/fpp3/complexseasonality.html). 
For this model, I do not choose SARIMA, because it would not properly represent the complex seasonal patterns. 


## ACF/PACF

``` r
occ_ts %>% diff() %>% ggAcf(main='ACF') 
occ_ts %>% diff() %>% ggPacf(main='PACF')
```

<iframe src="{{ 'assets/img/ACF.png' | relative_url }}" type="image" width="100%" height="400"></iframe>
<iframe src="{{ 'assets/img/PACF.png' | relative_url }}" type="image" width="100%" height="400"></iframe>

In the ACF, there are significant spikes at lags 12, 24, and 36 before
fully tapering off. In the PACF there is a large negative spike at lag
2. There is also a significant at lag 24 before tapering off.

The ACF and PACF indicate MA(1), AR(2), with one difference needed to
make the series stationary. Therefore, the ARIMA order would logically be (2,1,1).
The auto arima model's resulting fit corroborates this.

``` r
auto.arima(occ_ts)
```

    ## Series: occ_ts 
    ## ARIMA(2,1,1) with drift 
    ## 
    ## Coefficients:
    ##          ar1      ar2      ma1   drift
    ##       1.6840  -0.7700  -0.9827  0.0092
    ## s.e.  0.0054   0.0053   0.0019  0.2170
    ## 
    ## sigma^2 = 16534:  log likelihood = -90272.95
    ## AIC=180555.9   AICc=180555.9   BIC=180593.8


## ARIMA Forecast

``` r
# ignoring seasonal component?
fit <- Arima(occ_ts, order=c(2,1,1), lambda = occLam)
# checkresiduals(fit)
fit %>% forecast(h=24*14) %>% autoplot()
```

<iframe src="{{ 'assets/img/ARIMA.png' | relative_url }}" type="image" width="100%" height="400"></iframe>

``` r
accuracy(fit)
```

    ##                    ME     RMSE      MAE MPE MAPE      MASE        ACF1
    ## Training set 6.347566 130.2014 76.56732 NaN  Inf 0.5912514 -0.05623606

This particular ARIMA model doesn’t accurately capture the trend. The
hourly frequency coupled with the repeated patterns over large periods
of time makes the seasonality of the series complex.
[Hyndman](https://stackoverflow.com/questions/70917360/how-to-specify-annual-seasonality-for-an-arima-model-of-fable)
suggests using [Fourier Regression](https://otexts.com/fpp3/complexseasonality.html#example-electricity-demand)
to represent the complex seasonality. Adding Fourier regressors accounts
for complex seasonality and makes the model more accurate.

## Fourier

In order to avoid letting the forecast predict negative values, I chose
to take the square root of the predictor variable *average_occupancy* in
model selection. I chose 0.5 because it was closest to the auto selected
lamda parameter that didn’t cause the model to fall outside the unit
circle. The models with week period fourier regressors accurately
capture the scope of the summer quarter as well as the cyclical peaks.

The *K* Fourier terms are selected with AIC criterion and arbitrary
selection. The AIC values were smallest with larger period increments
for year and day. The final model was chosen based on the smallest AIC
value, RSME value, and the best visual fit.

``` r
# K breaks after 14 
# increasing days past 6/7 doesn't make a difference visually 
fit_select <- model(floor.clean,
   `K=6 day, K=2 week, K=1 year` = ARIMA(sqrt(average_occupancy) ~ 
           fourier(period='year', K=1) + 
           fourier(period='week', K=2) + 
           fourier(period='day', K=6) +
            PDQ(0,0,0) + pdq(d = 0)),
  `K=7 day, K=2 week, K=1 year` = ARIMA(sqrt(average_occupancy) ~ 
           fourier(period='year', K=1) + 
           fourier(period='week', K=2) + 
           fourier(period='day', K=7) + 
           PDQ(0,0,0) + pdq(d = 0))
)
```

## [Evaluate model accuracy](https://otexts.com/fpp2/accuracy.html)

``` r
glance(fit_select) # K=2wk is best 
```

    ## # A tibble: 2 × 8
    ##   .model                   sigma2 log_lik    AIC   AICc    BIC ar_roots ma_roots
    ##   <chr>                     <dbl>   <dbl>  <dbl>  <dbl>  <dbl> <list>   <list>  
    ## 1 K=6 day, K=2 week, K=1 …   3.46 -29735. 59513. 59513. 59681. <cpl>    <cpl>   
    ## 2 K=7 day, K=2 week, K=1 …   3.43 -29674. 59397. 59397. 59588. <cpl>    <cpl>

``` r
accuracy(fit_select)
```

    ## # A tibble: 2 × 10
    ##   .model                  .type    ME  RMSE   MAE   MPE  MAPE  MASE RMSSE   ACF1
    ##   <chr>                   <chr> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl>  <dbl>
    ## 1 K=6 day, K=2 week, K=1… Trai…  3.70  97.0  58.0  -Inf   Inf 0.298 0.290 -0.198
    ## 2 K=7 day, K=2 week, K=1… Trai…  3.67  96.7  57.9  -Inf   Inf 0.297 0.289 -0.185

``` r
fit_select |>
  forecast(h = 24*14) |>
  autoplot(floor.clean, level = 95) +
  facet_wrap(vars(.model), ncol = 2) +
  guides(colour = "none", fill = "none", level = "none") 
```

<iframe src="{{ 'assets/img/ARIMA_model_select_1.png' | relative_url }}" type="image" width="100%" height="400"></iframe>
<iframe src="{{ 'assets/img/ARIMA_model_select_2.png' | relative_url }}" type="image" width="100%" height="400"></iframe>

``` r
# fit_select %>% forecast(h=24*14) %>% autoplot()
```

Using Fourier regressors makes the forecast more cyclical and better representative of the 24 hour periods.

### AIC Selection

``` r
fit <- floor.clean %>% 
    as_tsibble(., index=timestamp) %>% 
    model(
      ARIMA(sqrt(average_occupancy) ~ 
            PDQ(0, 0, 0) + pdq(d = 0) +
            fourier(period = "day",  K = 6) +
            fourier(period = "week",  K = 2) + 
            fourier(period = "year", K = 1) 
      )
    )

### AIC Selection 
# report(fit)
```

## [Train and Test](https://otexts.com/fpp3/training-test.html)

``` r
# floor.clean[-7*24*12,]
slice(floor.clean, 15200-7*24*11)
```

    ## # A tsibble: 1 x 3 [1h] <UTC>
    ##   timestamp           average_occupancy peak_occupancy
    ##   <dttm>                          <dbl>          <dbl>
    ## 1 2023-04-01 23:00:00                10             11

``` r
# nrow(floor.clean): 15200
# larger k terms result in a less accurate fit 
train <- floor.clean |> filter(timestamp <= "2023-04-01 23:00:00")
test <- floor.clean |> filter(timestamp > "2023-04-01 23:00:00")

floor.train <- train %>% 
    as_tsibble(., index=timestamp) %>% 
    model(
      ARIMA(sqrt(average_occupancy) ~ 
            PDQ(0, 0, 0) + pdq(d = 0) +
            fourier(period = "day",  K = 7) +
            fourier(period = "week",  K = 2) + 
            fourier(period = "year", K = 1) 
      )
    )
```

``` r
floor.train |>
  forecast(h = 7*24*11) |>
  autoplot(floor.clean)
```

<iframe src="{{ 'assets/img/ARIMA_model_1.png' | relative_url }}" type="image" width="100%" height="400"></iframe>

``` r
floor.train |>
  forecast(h = 7*24*11) |>
  autoplot(test)
```

<iframe src="{{ 'assets/img/ARIMA_model_2.png' | relative_url }}" type="image" width="100%" height="400"></iframe>
Aside from the areas where the region
is supposed to be low during breaks, the fit appears to be relatively
accurate. 

*References*
+ <https://people.duke.edu/~rnau/411arim3.htm> 
+ <https://stats.stackexchange.com/questions/378347/selecting-arima-p-d-q-paramerters-for-hourly-data-with-24-hour-cycle>
+ <https://stats.stackexchange.com/questions/16117/what-method-can-be-used-to-detect-seasonality-in-data>
+ <https://stats.stackexchange.com/questions/221411/how-to-deal-with-hourly-non-stationary-time-series-data-with-multi-seasonality>  
+ <https://otexts.com/fpp2/> <https://otexts.com/fpp3/>
