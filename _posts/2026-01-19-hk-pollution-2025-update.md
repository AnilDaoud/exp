---
layout: post
title: "Hong Kong Pollution Data - 2025 Update"
date: 2026-01-19
tags: experiments jupyter HK hongkong pollution charts aqi python
---

It's been over 8 years since my [original analysis of Hong Kong pollution data](https://anil.diwi.org/exp/2018/11/04/hk-pollution-deep-dive/). Time for an update! Has air quality improved? What trends can we observe now with almost 27 years of data (1998-2025)?

In this post, we'll:
1. Update the scraping tool to work with the current EPD website
2. Download fresh data for the Central/Western district
3. Analyze long-term trends and confirm that pollution is decreasing

Data downloaded from [HK EPD](https://cd.epic.epd.gov.hk/EPICDI/air/station/?lang=en).

## Part 1: Updating the Scraper

The original Selenium-based scraper from 2018 no longer works - the EPD website has been completely redesigned and now uses Jakarta Server Faces (JSF). I've rewritten the scraper using Playwright, which handles the dynamic JavaScript forms better.

Key changes:
- Switched from Selenium to Playwright (headless mode, works over SSH)
- Handle JSF form submissions with hidden elements
- Download "By Station" format (wide format with all pollutants as columns)

```python
# Install dependencies
# pip install playwright pandas numpy python-aqi
# playwright install chromium

# Download data for Central/Western station (1998-2025)
python hk_pollution_scraper.py \
    --stations central_western \
    --time-range hourly \
    --from 1998-01-01 \
    --to 2025-01-19 \
    --output central-western
```

The scraper automatically:
- Downloads yearly CSV files from EPD
- Converts units to EPA standard (CO to ppm, O3 to ppm, NO2/SO2 to ppb)
- Calculates 8-hour rolling averages for CO and O3
- Computes AQI using the [python-aqi](https://github.com/hrbonz/python-aqi) library
- Saves both hourly and daily AQI data

Please leave a comment if you need the script, i'll upload it somewhere

## Part 2: Loading the Data

```python
import pandas as pd
import os

folder = 'central-western'
df = pd.read_csv(os.path.join(os.getcwd(), folder, 'aqi_hourly_full.csv'))
df.drop(df.columns[[0]], axis=1, inplace=True)
df.DATE = pd.to_datetime(df.DATE, format="%Y-%m-%d")
df.HOUR = pd.to_timedelta(df.HOUR, unit='h')
df['TIMESTAMP'] = df.DATE + df.HOUR
df.iloc[:,2:10] = df.iloc[:,2:10].apply(pd.to_numeric, errors='coerce')
dailyaqi = df.iloc[:,[0,9]].groupby(['DATE']).median().copy().T.squeeze()
hourlyaqi = df.loc[:,['TIMESTAMP','AQI']].groupby(['TIMESTAMP']).median().copy().T.squeeze()
```

## Part 3: 27 Years of Hong Kong Air Quality

Let's start with the full daily AQI time series from 1998 to 2025:

```python
dailyaqi.plot(figsize=(16,6))
```

![Daily AQI 1998-2025](../../../../assets/hkaqi2025/output_2_1.png)

Several things stand out:
- The massive spike around 2010 reaching AQI ~500 was the [infamous Asian dust storm](http://news.bbc.co.uk/2/hi/asia-pacific/8579495.stm) from Mongolia/China
- There's a clear regime change around 2011-2012 when PM2.5 monitoring began
- Recent years (2020-2025) show noticeably lower peaks

## What's Driving the AQI?

Let's look at which pollutant is the primary driver of AQI each year:

```python
aqi_origins = df.groupby(['DATE']).median().copy()
aqi_origins['AQI'] = aqi_origins.iloc[:,:6].max(numeric_only=True, axis=1)
aqi_origins['AQISource'] = aqi_origins.loc[:,['FSP','NO2','O3','RSP','SO2','CO_8h','O3_8h']].eq(
    aqi_origins['AQI'], axis=0).apply(
    lambda x: ','.join(aqi_origins.columns[0:7][x==x.max()]), axis=1)
aqi_origins.loc[aqi_origins['AQI'].isnull(),'AQISource'] = pd.np.nan
aqi_origins = aqi_origins.groupby(aqi_origins.index.year).apply(
    lambda x: x['AQISource'].value_counts(normalize=True))
mask = aqi_origins > 0.025
aqi_origins = aqi_origins.loc[mask]
aqi_origins = aqi_origins.unstack().fillna(0)
aqi_origins.plot.bar(stacked=True, figsize=(16,8))
```

![AQI Source by Year](../../../../assets/hkaqi2025/output_3_2.png)

The story is clear:
- **1998-2010**: RSP (PM10) was the main driver, with some NO2 contribution
- **2011**: Transition year - mix of RSP, NO2, and the new FSP (PM2.5) measurement
- **2012-2025**: FSP (PM2.5) completely dominates as the primary pollutant

This transition coincides with Hong Kong starting to measure PM2.5 (Fine Suspended Particulates) around 2011-2012. PM2.5 wasn't being measured before, so it couldn't be the AQI driver even if it was present in the air.

## Year-by-Year Distribution

Box plots show the distribution of daily AQI values per year:

```python
groups = dailyaqi.groupby(dailyaqi.index.year)
groupstack = pd.concat([pd.DataFrame(x[1].values) for x in groups], axis=1)
groupstack.columns = [str(x) for x in groups.groups.keys()]
groupstack.boxplot(figsize=(16,6))
```

![Yearly Boxplots](../../../../assets/hkaqi2025/output_4_1.png)

The 2010 outlier is clearly visible. More importantly, the median AQI (green line) and the interquartile range both appear to be declining in recent years.

## The Last Decade: 2013-2025

Let's zoom in on recent years:

```python
dailyaqi['2013':].plot(style='k.', figsize=(16,6))
```

![Recent Years Scatter](../../../../assets/hkaqi2025/output_5_1.png)

The downward trend becomes more apparent. The ceiling of daily values has clearly dropped - we're seeing fewer days above AQI 150, and many more days in the "Good" range (below 50).

## Seasonality: Heatmap View

A heatmap of the last 8 years (2018-2025) by day of year:

![Yearly Heatmap](../../../../assets/hkaqi2025/output_6_2.png)

Winter months (left side, days 0-90 and right side, days 300-365) consistently show higher pollution (yellow/green), while summer months (days 150-250, roughly June-September) are cleaner (dark purple).

## Weekly Seasonality

```python
weeks = dailyaqi['2018':].groupby(dailyaqi['2018':].index.isocalendar().week)
weekstack = pd.concat([pd.DataFrame(x[1].values) for x in weeks], axis=1)
weekstack.columns = [str(x) for x in weeks.groups.keys()]
weekstack.boxplot(figsize=(16,6))
```

![Weekly Boxplots](../../../../assets/hkaqi2025/output_7_2.png)

Weeks 25-35 (late June through early September) have the lowest pollution - this is the summer monsoon season when southerly winds bring cleaner maritime air. Winter weeks (1-10 and 45-52) see higher pollution from northerly winds carrying continental pollution.

## Monthly Pattern

```python
months = dailyaqi['2018':].groupby(dailyaqi['2018':].index.month)
monthstack = pd.concat([pd.DataFrame(x[1].values) for x in months], axis=1)
monthstack.columns = [str(x) for x in months.groups.keys()]
monthstack.boxplot(figsize=(12,6))
```

![Monthly Boxplots](../../../../assets/hkaqi2025/output_8_1.png)

The pattern is even clearer by month:
- **January-March**: Higher pollution (median ~70-75)
- **June-September**: Cleanest months (median ~25-30)
- **October-December**: Rising again

This aligns with Hong Kong's monsoon patterns - summer southwest monsoons bring clean oceanic air, while winter northeast monsoons bring pollution from mainland China.

---

## Part 4: Additional Analysis

### Long-Term Trend: Average AQI by Year

```python
yearly_avg = dailyaqi.groupby(dailyaqi.index.year).mean()
yearly_avg.plot(kind='bar', figsize=(14,4), title='Average AQI by Year', color='steelblue')
plt.axhline(y=yearly_avg.mean(), color='red', linestyle='--', label=f'Overall mean: {yearly_avg.mean():.1f}')
```

![Average AQI by Year](../../../../assets/hkaqi2025/output_10_1.png)

The bar chart clearly shows:
- **1998-2010**: Consistently low AQI (35-50 range) - but this is misleading as PM2.5 wasn't measured
- **2011-2015**: Spike to 60-90 after PM2.5 monitoring began
- **2016-2025**: Gradual decline back toward 50

### Rolling Averages: The True Trend

```python
dailyaqi.rolling(365).mean().plot(label='365-day rolling mean', linewidth=2)
dailyaqi.rolling(30).mean().plot(alpha=0.3, label='30-day rolling mean')
```

![Rolling Averages](../../../../assets/hkaqi2025/output_10_2.png)

The 365-day rolling average (blue line) tells the complete story:
- Stable around 45 from 1998-2010
- Sharp rise to peak of ~90 around 2013 (after PM2.5 monitoring)
- Steady decline since 2014, now back to ~50

### Hour of Day Pattern

```python
hourly_pattern = df.groupby('HOUR')['AQI'].median()
hourly_pattern.plot(kind='bar', figsize=(12,4), title='Median AQI by Hour of Day')
```

![Hourly Pattern](../../../../assets/hkaqi2025/output_11_0.png)

Interestingly, there's not a dramatic rush-hour spike at Central/Western (a general station, not roadside). The pattern shows:
- **Lowest**: Early morning (3-7 AM), around AQI 44-45
- **Highest**: Afternoon/evening (3-9 PM), around AQI 54-55
- Difference of only ~10 AQI points

This suggests the pollution at this station is more influenced by regional/background levels than local traffic.

### Day of Week Pattern

```python
df['DOW'] = df['TIMESTAMP'].dt.dayofweek
dow_aqi = df.groupby('DOW')['AQI'].median()
```

![Day of Week](../../../../assets/hkaqi2025/output_12_0.png)

Surprisingly, there's almost no weekday/weekend difference:
- **Weekday average**: 51
- **Weekend average**: 51.5

This reinforces that the Central/Western station measures regional pollution rather than local traffic emissions. Roadside stations like Causeway Bay would show a bigger difference.

### COVID-19 Impact: 2019 vs 2020 vs 2021

```python
comparison = pd.DataFrame({
    '2019': monthly_2019.values,
    '2020': monthly_2020.values,
    '2021': monthly_2021.values
}, index=['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'])
comparison.plot(kind='bar')
```

![COVID Impact](../../../../assets/hkaqi2025/output_13_0.png)

COVID-19 impact is visible:
- **2019 median AQI**: 66.0
- **2020 median AQI**: 59.0
- **2021 median AQI**: 56.0

The reduction during 2020-2021 lockdowns is clear, especially in winter months when the difference is most pronounced. Summer months show less difference as they're already clean due to monsoon patterns.

### Exceedance Days: How Often is Air "Unhealthy"?

```python
thresholds = {
    'Unhealthy for Sensitive (>100)': 100,
    'Unhealthy (>150)': 150,
    'Very Unhealthy (>200)': 200
}
```

![Exceedance Days](../../../../assets/hkaqi2025/output_14_0.png)

This is perhaps the most striking chart:
- **2013 peak**: 137 days above AQI 100, 45+ days above 150
- **2020-2025**: Only 5-15 days above AQI 100, almost zero above 150

The number of "bad air days" has dropped by **90%** since the worst years.

### Seasonal Decomposition

Breaking down the time series into trend and seasonal components:

![Seasonal Decomposition](../../../../assets/hkaqi2025/output_15_0.png)

The three panels show:
1. **Top**: Raw monthly median AQI - the noisy signal
2. **Middle**: Trend component (12-month rolling average) - clearly shows the rise and fall
3. **Bottom**: Seasonal component - consistent winter peaks and summer troughs throughout the entire period

### Era Comparison

Grouping into distinct periods:

```python
decades = {
    '1998-2004': dailyaqi['1998':'2004'],
    '2005-2011': dailyaqi['2005':'2011'],
    '2012-2018': dailyaqi['2012':'2018'],
    '2019-2025': dailyaqi['2019':'2025']
}
```

![Era Comparison](../../../../assets/hkaqi2025/output_16_0.png)

| Era | Mean AQI | Median AQI |
|-----|----------|------------|
| 1998-2004 | 43 | 43 |
| 2005-2011 | 46 | 44 |
| 2012-2018 | 76 | 71 |
| 2019-2025 | 53 | 52 |

The 2012-2018 period (post-PM2.5 monitoring) was the worst, but 2019-2025 shows significant improvement - AQI has dropped 30% from the 2012-2018 peak.

---

## The Key Finding: Statistically Significant Improvement

Focusing on 2016-2025 (after PM2.5 monitoring stabilized):

```python
from scipy import stats
yearly_avg_recent = yearly_avg[yearly_avg.index >= 2016]
slope, intercept, r_value, p_value, std_err = stats.linregress(years, yearly_avg_recent.values)
```

![Trend 2016-2025](../../../../assets/hkaqi2025/output_17_1.png)

**Statistical Results:**
- **Trend**: -2.82 AQI per year
- **R-squared**: 0.89 (very strong fit)
- **P-value**: 0.000 (highly significant)

**Period Comparison:**
- **2016-2020 average**: 64.4 AQI
- **2021-2025 average**: 49.7 AQI
- **Improvement**: 14.7 points (**23% reduction**)

---

## Conclusions

After 27 years of data and comprehensive analysis:

1. **PM2.5 dominates since 2012** - Fine Suspended Particulates drive nearly 100% of high AQI days

2. **Strong, consistent seasonality** - Summer (June-September) is ~60% cleaner than winter due to monsoon patterns

3. **No significant weekday/weekend or rush-hour effect** - Central/Western measures regional pollution, not local traffic

4. **COVID-19 had measurable impact** - 10-15% reduction in AQI during 2020-2021

5. **90% fewer "bad air days"** - Days exceeding AQI 100 dropped from 137 (2013) to under 15 (recent years)

6. **Statistically significant improvement since 2016** - AQI declining by 2.8 points per year, with 23% overall reduction

The improvement is likely due to:
- Stricter emissions controls in Hong Kong
- Reduced industrial activity in the Pearl River Delta
- Cleaner vehicle fleet (more EVs, tighter emission standards)
- Some effect from reduced activity during COVID-19

**Hong Kong's air quality is genuinely improving.** The data shows real, measurable, statistically significant progress over the last decade. While PM2.5 remains the primary concern, the trend is encouraging.

---

*Code and data available on [GitHub](https://github.com/anildaoud). Updated scraper works with the 2025 EPD website.*
