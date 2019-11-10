---
layout: post
title:  "Scraping Hong Kong pollution data - Part 2"
tags: experiments jupyter HK hongkong pollution aqi
---

Pollution is a major concern in Hong Kong. How bad is it? Compared to say  5 years ago? 10 years ago?
It is surprisingly difficult to find simple long term charts, despite pollution data being seemingly readily available in a few places on the web.

[Part 1](https://anil.diwi.org/exp/2018/10/28/scraping-hk-pollution-1/) we collected some data from a HK Government website.

In this part we'll calculate the AQI using US EPA standard

In the next post let's try to explore the components of AQI to see where the pollution in HK comes from, if there is any seasonality in the data, if holidays have any impact and what times of the day are the worst.

Data downloaded from [HK EPD](https://cd.epic.epd.gov.hk/EPICDI/air/station/?lang=en), see [Part 1](https://anil.diwi.org/exp/2018/10/28/scraping-hk-pollution-1/). We use data for Causeway Bay in this post too.

[API calculation reference](https://airnow.gov/sites/default/files/2018-09/aqi-technical-assistance-document-sept-2018_0.pdf)

Using implementation in [python-aqi](https://github.com/hrbonz/python-aqi)

Remarks:
1. All Pollutant unit in μg/m3 except CO which is in 10μg/m3
2. N.A. = data not available
3. CO = Carbon Monoxide
4. FSP = Fine Suspended Particulates 
5. NO2 = Nitrogen Dioxide
6. NOX = Nitrogen Oxides
7. O3 = Ozone
8. RSP = Respirable Suspended Particulates
9. SO2 = Sulphur Dioxide



```python
import pandas as pd
import aqi
import os, sys, glob
from tqdm import tnrange, tqdm_notebook
```


```python
# what to import
folder = 'cwb'
if os.path.isfile(os.path.join(os.getcwd(),folder,'aqi_hourly_full.csv')):
    data_ready = True
    df = pd.read_csv(os.path.join(os.getcwd(),folder,'aqi_hourly_full.csv'))
    df.drop(df.columns[[0]], axis=1, inplace=True)
else:
    data_ready = False
    dfs = (pd.read_csv(f,skiprows=11) for f in sorted(glob.glob(os.path.join(os.getcwd(),folder,'air_hourly*.csv'))))
    df = pd.concat(dfs, ignore_index=True)
    del df['NOX']
    del df['STATION']
if data_ready:
    df.DATE = pd.to_datetime(df.DATE, format="%Y-%m-%d")
else:
    df.DATE = pd.to_datetime(df.DATE, format="%d/%m/%Y")
df.iloc[:,2:] = df.iloc[:,2:].apply(pd.to_numeric, errors='coerce')
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>DATE</th>
      <th>HOUR</th>
      <th>CO</th>
      <th>FSP</th>
      <th>NO2</th>
      <th>O3</th>
      <th>RSP</th>
      <th>SO2</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1998-01-01</td>
      <td>1</td>
      <td>203.0</td>
      <td>NaN</td>
      <td>116.0</td>
      <td>NaN</td>
      <td>107.0</td>
      <td>19.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1998-01-01</td>
      <td>2</td>
      <td>157.0</td>
      <td>NaN</td>
      <td>119.0</td>
      <td>NaN</td>
      <td>99.0</td>
      <td>15.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1998-01-01</td>
      <td>3</td>
      <td>156.0</td>
      <td>NaN</td>
      <td>127.0</td>
      <td>NaN</td>
      <td>112.0</td>
      <td>14.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1998-01-01</td>
      <td>4</td>
      <td>179.0</td>
      <td>NaN</td>
      <td>114.0</td>
      <td>NaN</td>
      <td>103.0</td>
      <td>17.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1998-01-01</td>
      <td>5</td>
      <td>156.0</td>
      <td>NaN</td>
      <td>130.0</td>
      <td>NaN</td>
      <td>112.0</td>
      <td>58.0</td>
    </tr>
  </tbody>
</table>
</div>



AQI with EPA calculation method requires these units:
pm10 (µg/m³), o3_8h (ppm), co_8h (ppm), no2_1h (ppb), o3_1h (ppm), so2_1h (ppb), pm25 (µg/m³)
[Conversion factors](https://uk-air.defra.gov.uk/assets/documents/reports/cat06/0502160851_Conversion_Factors_Between_ppb_and.pdf)


```python
if not data_ready:
    # CONVERT CO FROM 10 µg/m3 to ppm
    # 1 ppm = 1.1642 mg m-3 -> 1 ppm = (1.1642 * 1000) µg/m3
    df.CO = df.CO / (1.1642 * 100)
    # CONVERT O3 from µg/m3 to ppm
    # 1 ppb = 1.9957 µg m-3 -> 1 ppm = 1000 ppb = (1.9957 * 1000) µg m-3
    df['O3'] = df.O3/(1000 * 1.9957)
    # CONVERT NO2 from µg/m3 to ppb
    # 1 ppb = 1.9125 µg m-3
    df['NO2'] = df.NO2 / 1.9125
    # CONVERT SO2 from µg/m3 to ppb
    # 1 ppb = 2.6609 µg m-3
    df['SO2'] = df.NO2 / 2.6609
    # CALCULATE 8h moving averages
    df['CO_8h'] = df.CO.rolling(window=8).mean()
    df['O3_8h'] = df.O3.rolling(window=8).mean()
    # drop CO
    del df['CO']
```

Let's define two helper structures to match our data set with the constants from python-aqi


```python
index = {'CO_8h': aqi.POLLUTANT_CO_8H,
        'FSP' : aqi.POLLUTANT_PM25,
        'NO2' : aqi.POLLUTANT_NO2_1H,
        'RSP' : aqi.POLLUTANT_PM10,
        'SO2' : aqi.POLLUTANT_SO2_1H,
        'O3' : aqi.POLLUTANT_O3_1H,
        'O3_8h' : aqi.POLLUTANT_O3_8H}

caps = {'CO_8h': 50.4,
        'FSP' : 500.4,
        'NO2' : 2049,
        'RSP' : 604,
        'SO2' : 1004,
        'O3' : 0.604,
        'O3_8h' : 0.374}
```

Now we can apply AQI calculations on our DataFrame


```python
def aqi_atom(cc, cap, p):
    try:
        if pd.isnull(cc):
            return None
        else:
            return float(aqi.to_aqi([(p,min(cap,cc))]))
    except (ZeroDivisionError, IndexError) as e:
        return 0

aqidf = df.copy()
if not data_ready:
    for i in tnrange(2,df.shape[1]):
        p = index[df.columns[i]]
        cap = caps[df.columns[i]]
        aqidf.iloc[:,i] = df.iloc[:,i].apply(aqi_atom,args=(cap,p))
    aqi_series = aqidf.iloc[:,2:].max(numeric_only=True, axis = 1)
    aqi_series.name = "AQI"
    aqidf = aqidf.join(aqi_series)
dailyaqi = aqidf.iloc[:,[0,9]].groupby(['DATE']).median().copy().T.squeeze()
aqidf.head()
```
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>DATE</th>
      <th>HOUR</th>
      <th>FSP</th>
      <th>NO2</th>
      <th>O3</th>
      <th>RSP</th>
      <th>SO2</th>
      <th>CO_8h</th>
      <th>O3_8h</th>
      <th>AQI</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1998-01-01</td>
      <td>1</td>
      <td>NaN</td>
      <td>57.0</td>
      <td>NaN</td>
      <td>77.0</td>
      <td>31.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>77.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1998-01-01</td>
      <td>2</td>
      <td>NaN</td>
      <td>60.0</td>
      <td>NaN</td>
      <td>73.0</td>
      <td>33.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>73.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1998-01-01</td>
      <td>3</td>
      <td>NaN</td>
      <td>64.0</td>
      <td>NaN</td>
      <td>79.0</td>
      <td>34.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>79.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1998-01-01</td>
      <td>4</td>
      <td>NaN</td>
      <td>56.0</td>
      <td>NaN</td>
      <td>75.0</td>
      <td>31.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>75.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1998-01-01</td>
      <td>5</td>
      <td>NaN</td>
      <td>65.0</td>
      <td>NaN</td>
      <td>79.0</td>
      <td>36.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>79.0</td>
    </tr>
  </tbody>
</table>
</div>



And we can save the resulting timeseries in nice CSV format for later use


```python
if not data_ready:
    dailyaqi.to_csv(folder + '/aqi_daily.csv')
    aqidf.to_csv(folder + '/aqi_hourly_full.csv')
```

