# SQL Prophet

Forecasting and fitting with SQL

<Grid cols=2>
<Dropdown name=components title="Forecast Components" order=false>
    <DropdownOption value=trend valueLabel=Linear/>
    <DropdownOption value=yearly valueLabel=Yearly/>
    <DropdownOption value=quarterly valueLabel=Quarterly/>
    <DropdownOption value=monthly valueLabel=Monthly/>
</Dropdown>

<Slider title="Forecast Years" defaultValue=0 name=forecast_years max=5/>
</Grid>

<TextInput name=starting_balance title="Starting Cash Balance" defaultValue=55000/>

<TextInput name=burn title="Monthly Burn" defaultValue=10000/>



```sql weekly_sales
with weekly_sales as (
    select 
    date_trunc('week', order_datetime) as ds,
    sum(sales) as y
    from orders
    group by all
),

future_dates as (
    select unnest(generate_series(
        (select min(ds) from weekly_sales),
        (select max(ds) from weekly_sales) + interval '${inputs.forecast_years} years',
        interval '1 week'
    )) as ds
)

select 
    f.ds,
    w.y
from future_dates f
left join weekly_sales w
on f.ds = w.ds
```


```sql fourier_terms
-- step 1: calculate fourier terms
with fourier_terms as (
    select ds,
            y,
            row_number() over (order by ds) as time_index,
            -- yearly
            sin(2 * pi() * extract(weekofyear from ds) / 52) as sin1,
            cos(2 * pi() * extract(weekofyear from ds) / 52) as cos1,
            -- quarterly
            sin(8 * pi() * extract(weekofyear from ds) / 52) as sin4,
            cos(8 * pi() * extract(weekofyear from ds) / 52) as cos4,
            -- monthly
            sin(24 * pi() * extract(weekofyear from ds) / 52) as sin12,
            cos(24 * pi() * extract(weekofyear from ds) / 52) as cos12,

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
        regr_slope(y, sin1) as beta_sin1,
        regr_slope(y, cos1) as beta_cos1,
        regr_slope(y, sin4) as beta_sin4,
        regr_slope(y, cos4) as beta_cos4,
        regr_slope(y, sin12) as beta_sin12,
        regr_slope(y, cos12) as beta_cos12,
        avg(sin1) as mean_sin1,
        avg(cos1) as mean_cos1,
        avg(sin4) as mean_sin4,
        avg(cos4) as mean_cos4,
        avg(sin12) as mean_sin12,
        avg(cos12) as mean_cos12,
    from ${fourier_terms}
)
select * from coefficients
```


```sql sales_fit
-- step 3: predict using the selected coefficients
with predictions as (
    select 
        f.ds,
        f.y,
        c.trend_intercept + c.trend_slope * f.time_index as trend_yhat,
        case '${inputs.components.value}'
            when 'trend' then 0
            when 'yearly' then 
            c.beta_sin1 * (f.sin1 - c.mean_sin1) + c.beta_cos1 * (f.cos1 - c.mean_cos1) 
            when 'quarterly' then 
            c.beta_sin1 * (f.sin1 - c.mean_sin1) + c.beta_cos1 * (f.cos1 - c.mean_cos1) 
            + c.beta_sin4 * (f.sin4 - c.mean_sin4) + c.beta_cos4 * (f.cos4 - c.mean_cos4)
            when 'monthly' then
            c.beta_sin1 * (f.sin1 - c.mean_sin1) + c.beta_cos1 * (f.cos1 - c.mean_cos1) 
            + c.beta_sin4 * (f.sin4 - c.mean_sin4) + c.beta_cos4 * (f.cos4 - c.mean_cos4)
            + c.beta_sin12 * (f.sin12 - c.mean_sin12) + c.beta_cos12 * (f.cos12 - c.mean_cos12)
        end as seasonality_yhat
    from ${fourier_terms} f, ${coefficients} c
)

-- final step: combine trend and seasonality effects
select 
    ds,
    y as historic,
    trend_yhat as trend,
    trend_yhat + seasonality_yhat as trend_plus_seasonality,
    case 
        when ds >= '2021-12-31' then trend_yhat + seasonality_yhat 
        else null 
        end as forecast,
    ${inputs.starting_balance}::double + sum(coalesce(historic, trend_plus_seasonality) - ${inputs.burn}::double/4) over (order by ds) as cash_balance,
from predictions
```

<Grid cols=2>

<LineChart
  data={sales_fit}
  x=ds
  y={['historic','trend', 'trend_plus_seasonality']}
  yFmt=usd
  title="Sales by Week, Forecast"
  colorPalette={['#2269A4', '#44A0BF', '#ff7300']}
  renderer=svg
/>


<LineChart
  data={sales_fit}
  x=ds
  y=cash_balance
  yFmt=usd
  title="Cash Balance by Week, Forecast"
  legend
/>

</Grid>

```sql sales_by_year_forecast
select 
    date_part('year', ds)::varchar as year,
    sum(historic) as historic,
    sum(forecast) as forecast
from ${sales_fit}
where year >= '2018-01-01'
group by all
```

<BarChart
  data={sales_by_year_forecast}
  x=year
  y={['historic', 'forecast']}
  sort=false
  colorPalette={['#2269A4', '#ff7300']}
  title="Sales by Year, Forecast"
/>