---
layout: post
title:  "Hong Kong pollution data"
tags: experiments jupyter HK hongkong pollution charts aqi
---

Pollution is a major concern in Hong Kong. How bad is it? Compared to say  5 years ago? 10 years ago?
It is surprisingly difficult to find simple long term charts, despite pollution data being seemingly readily available in a few places on the web.

[Part 1](https://anil.diwi.org/exp/2018/10/28/scraping-hk-pollution-1/) we collected some data from a HK Government website.

[Part 2](https://anil.diwi.org/exp/2018/10/30/scraping-hk-pollution-2/) we calculated the AQI manually.

In this let's try to explore the components of AQI to see where the pollution in HK comes from, if there is any obvious seasonality in the data, if holidays have any impact and what times of the day are the worst.

Data downloaded from [HK EPD](https://cd.epic.epd.gov.hk/EPICDI/air/station/?lang=en), see [Part 1](https://anil.diwi.org/exp/2018/10/28/scraping-hk-pollution-1/).

In order to mix it up a little, we use data for CENTRAL / Western district here. It should be less noisy as Causeway Bay data as it is not a roadside station.

Remarks:
3. CO = Carbon Monoxide
4. FSP = Fine Suspended Particulates, PM2.5 
5. NO2 = Nitrogen Dioxide
6. NOX = Nitrogen Oxides
7. O3 = Ozone
8. RSP = Respirable Suspended Particulates, PM10
9. SO2 = Sulphur Dioxide

Initial imports

```python
import pandas as pd
import os
```

Let's get the data from our previously saved CSV (see previous posts)

```python
folder = 'central-western'
df = pd.read_csv(os.path.join(os.getcwd(),folder,'aqi_hourly_full.csv'))
df.drop(df.columns[[0]], axis=1, inplace=True)
df.DATE = pd.to_datetime(df.DATE, format="%Y-%m-%d")
df.HOUR = pd.to_timedelta(df.HOUR, unit='h')
df['TIMESTAMP'] = df.DATE + df.HOUR
df.iloc[:,2:10] = df.iloc[:,2:10].apply(pd.to_numeric, errors='coerce')
dailyaqi = df.iloc[:,[0,9]].groupby(['DATE']).median().copy().T.squeeze()
hourlyaqi = df.loc[:,['TIMESTAMP','AQI']].groupby(['TIMESTAMP']).median().copy().T.squeeze()
```

and let's print a few plots to start


```python
dailyaqi.plot(figsize=(16,6))
```

![png](../../../../assets/hkaqi/output_4_1.png)


There seems to be a change in regime around 2011, 2012.

Let's try to pin down if this is due to a change in underlying AQI component.


```python
aqi_origins = df.groupby(['DATE']).median().copy()
aqi_origins['AQI'] = aqi_origins.iloc[:,:6].max(numeric_only=True, axis = 1)
aqi_origins['AQISource'] = aqi_origins.loc[:,['FSP','NO2','O3','RSP','SO2','CO_8h','O3_8h']].eq(aqi_origins['AQI'], axis=0).apply(lambda x: ','.join(aqi_origins.columns[0:7][x==x.max()]),axis=1)
aqi_origins.loc[aqi_origins['AQI'].isnull(),'AQISource'] = pd.np.nan
aqi_origins = aqi_origins.groupby(aqi_origins.index.year).apply(lambda x: x['AQISource'].value_counts(normalize=True))
# let's filter out small contributors to make the chart clearer
mask = aqi_origins>0.025
aqi_origins = aqi_origins.loc[mask]
aqi_origins = aqi_origins.unstack().fillna(0)
aqi_origins.plot.bar(stacked=True,figsize=(16,8))
```

![png](../../../../assets/hkaqi/output_6_1.png)


Indeed, before 1993 the main driver of AQI was NO2.

Between 1994 and 2011, it was PM10.

And since then, clearly PM2.5



Let's try to see box plots now because daily data for 20 years is not ideal.


```python
groups = dailyaqi.groupby(dailyaqi.index.year)
groupstack = pd.concat([pd.DataFrame(x[1].values) for x in groups], axis=1)
groupstack = pd.DataFrame(groupstack)
groupstack.columns = [str(x) for x in groups.groups.keys()]
groupstack.boxplot(figsize=(16,6))
```

![png](../../../../assets/hkaqi/output_8_1.png)


Let's zoom in on 2010 where it looks like we have a large outlier.

I thought it was a mistake at first, but then I found [this BBC News article online](http://news.bbc.co.uk/2/hi/asia-pacific/8579495.stm)


```python
dailyaqi['2010'].plot(figsize=(14,6))
```

![png](../../../../assets/hkaqi/output_10_1.png)


Enough with the old stuff, let's look at the last 5 years


```python
dailyaqi['2013':].plot(style='k.',figsize=(16,6))
```

![png](../../../../assets/hkaqi/output_12_1.png)


Is there seasonality? Let's plot the last 5 years to 2018 on a stacked basis


```python
groups = dailyaqi['2013':].groupby(pd.Grouper(freq='A'))
years = pd.DataFrame()
for name, group in groups:
    if group.values.shape[0] == 366:
        years[name.year] = group.drop([pd.to_datetime(str(name.year)+'/02/29', format='%Y/%m/%d')]).values
    elif group.values.shape[0] < 365:
        print('Year %s does not have enough data (%s points only), filling with zeros' % (str(name.year),str(group.values.shape[0])))
        years[name.year] = group.append(pd.Series([0] * (365-group.shape[0])),ignore_index=True).values
    elif group.values.shape[0] == 365:
        years[name.year] = group.values
    else:
        print('Year %s has too many data points (%s points!). Something is wrong with the data.' % (str(name.year),str(group.values.shape[0])))
        raise
years.plot(subplots=True, legend=False, figsize=(16,9))
```

    Year 2018 does not have enough data (212 points only), filling with zeros

![png](../../../../assets/hkaqi/output_14_2.png)


Not obvious, maybe with a heatmap we will see better?


```python
from matplotlib import pyplot
groups = dailyaqi['2013':].groupby(pd.Grouper(freq='A'))
years = pd.DataFrame()
for name, group in groups:
    if group.values.shape[0] == 366:
        years[name.year] = group.drop([pd.to_datetime(str(name.year)+'/02/29', format='%Y/%m/%d')]).values
    elif name.year == 2018:
        years[name.year] = group.append(pd.Series([0] * (365-group.shape[0])),ignore_index=True).values
    elif group.values.shape[0] == 365:
        years[name.year] = group.values
    else:
        raise
years = years.T
pyplot.matshow(years, interpolation=None, aspect='auto')
```

![png](../../../../assets/hkaqi/output_16_1.png)


So it looks like weaker numbers around the middle of the year, higher pollution around year end. Let's plot weekly boxes for last two years:


```python
sliceofdata = dailyaqi['2016':]
groups = sliceofdata.groupby(sliceofdata.index.weekofyear)
groupstack = pd.concat([pd.DataFrame(x[1].values) for x in groups], axis=1)
groupstack = pd.DataFrame(groupstack)
groupstack.columns = [str(x) for x in groups.groups.keys()]
groupstack.boxplot(figsize=(16,6))
```

![png](../../../../assets/hkaqi/output_18_1.png)


And now by month


```python
sliceofdata = dailyaqi['2016':]
groups = sliceofdata.groupby(sliceofdata.index.month)
groupstack = pd.concat([pd.DataFrame(x[1].values) for x in groups], axis=1)
groupstack = pd.DataFrame(groupstack)
groupstack.columns = [str(x) for x in groups.groups.keys()]
groupstack.boxplot(figsize=(16,6))
```

![png](../../../../assets/hkaqi/output_20_1.png)


Let's display monthly data from 2016:


```python
sliceofdata = dailyaqi['2016':]
groups = sliceofdata.groupby([sliceofdata.index.year,sliceofdata.index.month])
groupstack = pd.concat([pd.DataFrame(x[1].values) for x in groups], axis=1)
groupstack = pd.DataFrame(groupstack)
groupstack.columns = ['_'.join([str(y) for y in x]) for x in groups.groups.keys()]
groupstack.boxplot(figsize=(16,6))
```

![png](../../../../assets/hkaqi/output_22_1.png)


Bizarrely, December is when pollution seems to be the highest.

Now let's look at a particular month and see if there is a pattern during the day? Let's select Feb 2018, when median pollution was fairly high


```python
sliceofdata = hourlyaqi['2018-02']
groups = sliceofdata.groupby(sliceofdata.index.hour)
groupstack = pd.concat([pd.DataFrame(x[1].values) for x in groups], axis=1)
groupstack = pd.DataFrame(groupstack)
groupstack.columns = [str(x) for x in groups.groups.keys()]
groupstack.boxplot(figsize=(16,6))
```

![png](../../../../assets/hkaqi/output_24_1.png)


I don't see any pattern here.

Let's have a look at days of the week.


```python
sliceofdata = dailyaqi['2018-02']
groups = sliceofdata.groupby(sliceofdata.index.dayofweek)
groupstack = pd.concat([pd.DataFrame(x[1].values) for x in groups], axis=1)
groupstack = pd.DataFrame(groupstack)
groupstack.columns = [str(x) for x in groups.groups.keys()]
groupstack.boxplot(figsize=(16,6))
```

![png](../../../../assets/hkaqi/output_26_1.png)


No clear pattern here either
