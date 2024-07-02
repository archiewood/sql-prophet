# SQL Prophet

Forecasting and fitting with SQL

```sql weekly_sales
select 
  date_trunc('week', order_datetime) as ds,
  sum(sales) as y
from orders
group by all
```


```sql fourier_terms
-- step 1: calculate fourier terms
with fourier_terms as (
    select ds,
            y,
            row_number() over (order by ds) as time_index,
            sin(2 * pi() * extract(weekofyear from ds) / 52) as sin1,
            cos(2 * pi() * extract(weekofyear from ds) / 52) as cos1,
            sin(4 * pi() * extract(weekofyear from ds) / 52) as sin2,
            cos(4 * pi() * extract(weekofyear from ds) / 52) as cos2,
            sin(6 * pi() * extract(weekofyear from ds) / 52) as sin3,
            cos(6 * pi() * extract(weekofyear from ds) / 52) as cos3,
            sin(8 * pi() * extract(weekofyear from ds) / 52) as sin4,
            cos(8 * pi() * extract(weekofyear from ds) / 52) as cos4,
            sin(10 * pi() * extract(weekofyear from ds) / 52) as sin5,
            cos(10 * pi() * extract(weekofyear from ds) / 52) as cos5,
            sin(12 * pi() * extract(weekofyear from ds) / 52) as sin6,
            cos(12 * pi() * extract(weekofyear from ds) / 52) as cos6,
    from ${weekly_sales}
)
select * from fourier_terms
```

```sql coefficients
-- step 2: fit coefficients using linear regression
with coefficients as (
    select
        regr_intercept(y, time_index) as trend_intercept,
        regr_slope(y, time_index) as trend_slope,
        corr(y, sin1) * stddev_samp(y) / stddev_samp(sin1) as beta_sin1,
        corr(y, cos1) * stddev_samp(y) / stddev_samp(cos1) as beta_cos1,
        corr(y, sin2) * stddev_samp(y) / stddev_samp(sin2) as beta_sin2,
        corr(y, cos2) * stddev_samp(y) / stddev_samp(cos2) as beta_cos2,
        corr(y, sin3) * stddev_samp(y) / stddev_samp(sin3) as beta_sin3,
        corr(y, cos3) * stddev_samp(y) / stddev_samp(cos3) as beta_cos3,
        corr(y, sin4) * stddev_samp(y) / stddev_samp(sin4) as beta_sin4,
        corr(y, cos4) * stddev_samp(y) / stddev_samp(cos4) as beta_cos4,
        corr(y, sin5) * stddev_samp(y) / stddev_samp(sin5) as beta_sin5,
        corr(y, cos5) * stddev_samp(y) / stddev_samp(cos5) as beta_cos5,
        corr(y, sin6) * stddev_samp(y) / stddev_samp(sin6) as beta_sin6,
        corr(y, cos6) * stddev_samp(y) / stddev_samp(cos6) as beta_cos6,
        avg(y) as mean_y,
        avg(sin1) as mean_sin1,
        avg(cos1) as mean_cos1,
        avg(sin2) as mean_sin2,
        avg(cos2) as mean_cos2,
        avg(sin3) as mean_sin3,
        avg(cos3) as mean_cos3,
        avg(sin4) as mean_sin4,
        avg(cos4) as mean_cos4,
        avg(sin5) as mean_sin5,
        avg(cos5) as mean_cos5,
        avg(sin6) as mean_sin6,
        avg(cos6) as mean_cos6
    from ${fourier_terms}
)
select * from coefficients
```


```sql daily_sales_fit
-- step 3: predict using the fitted coefficients
with predictions as (
    select 
        f.ds,
        f.y,
        c.trend_intercept + c.trend_slope * f.time_index as trend_yhat,
        c.beta_sin1 * (f.sin1 - c.mean_sin1) + c.beta_cos1 * (f.cos1 - c.mean_cos1) 
        + c.beta_sin2 * (f.sin2 - c.mean_sin2) + c.beta_cos2 * (f.cos2 - c.mean_cos2)
        + c.beta_sin3 * (f.sin3 - c.mean_sin3) + c.beta_cos3 * (f.cos3 - c.mean_cos3)
        + c.beta_sin4 * (f.sin4 - c.mean_sin4) + c.beta_cos4 * (f.cos4 - c.mean_cos4)
        + c.beta_sin5 * (f.sin5 - c.mean_sin5) + c.beta_cos5 * (f.cos5 - c.mean_cos5)
        + c.beta_sin6 * (f.sin6 - c.mean_sin6) + c.beta_cos6 * (f.cos6 - c.mean_cos6)
         as seasonality_yhat
    from ${fourier_terms} f, ${coefficients} c
)

-- final step: combine trend and seasonality effects
select 
    ds,
    y,
    trend_yhat as trend_y_hat,
    trend_yhat + seasonality_yhat as y_hat
from predictions
```

<LineChart
  data={daily_sales_fit}
  x=ds
  y=y
/>


<LineChart
  data={daily_sales_fit}
  x=ds
  y={['y','trend_y_hat', 'y_hat']}
  colorPalette={['', '', '#ff7300']}
/>

<!-- 

```sql daily_sales_forecast
-- step 1: extrapolate future dates as a table
with future_dates as (
    select unnest(generate_series(
        (select min(ds) from ${daily_sales_fit}),
        (select max(ds) from ${daily_sales_fit}) + interval '1 year',
        interval '1 day'
    )) as ds
)
```

 -->