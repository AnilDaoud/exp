---
layout: post
title:  "Scraping Hong Kong pollution data - Part 1"
tags: experiments jupyter HK hongkong pollution scraping selenium
---

Pollution is a major concern in Hong Kong. How bad is it? Compared to say 5 years ago? 10 years ago?
It is surprisingly difficult to find simple charts of long term data, despite pollution data being seemingly readily available in a few places on the web.

Part 1 we'll collect some data.
Hopefully I'll have time for part 2 in the next few days (weeks?) to actually compute the AQI and display it. 

### AQICN.org

The folks over at [AQICN](https://aqicn.org/city/hongkong/) have a nifty web page, but you can only display a few days.
There is an [Open Data](http://aqicn.org/data-platform/register/) link, but you need to register to get historical data.

### Government data

The [Environmental Protection Department](https://www.epd.gov.hk/epd/epic/english/data_air.html) of Hong Kong has a bunch of data available for download. The starting point is [this page](https://cd.epic.epd.gov.hk/EPICDI/air/station/?lang=en): https://cd.epic.epd.gov.hk/EPICDI/air/station/?lang=en

There is a page somewhere where you can download excel files

The following is a quick script to get everything downloaded in CSV format. 

### Requirements

This uses selenium with chromedriver on Mac OS X



Import modules and set some variables


```python
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select
from dateutil import rrule
from datetime import datetime, timedelta
import calendar
import re, os, sys, time
from selenium.webdriver.chrome.options import Options
dlPath = '/Users/anil/Downloads/'
endPath = '/Users/anil/Projects/hk_aqi'
```

Open Chrome and Navigate to the initial page


```python
b = webdriver.Chrome()
b.get("https://cd.epic.epd.gov.hk/EPICDI/air/station/")
```

Click on Causeway Bay (for example)


```python
b.find_element_by_xpath('//*[@title="CAUSEWAY BAY"]').click()
```

Wait for page to load


```python
element_present = EC.presence_of_element_located((By.ID, 'form:timeRange:0'))
wait = WebDriverWait(b, 10).until(element_present)
```

Select hourly frequency


```python
rangeName = 'form:timeRange:0'
b.find_element_by_id(rangeName).click()
freq = b.find_element_by_xpath('//label[@for="'+rangeName+'"]').text.strip().lower()
dlFileName = 'air_'+freq+'.csv' # 'air_daily.csv if form:timeRange:1'
b.find_element_by_id("form:select").click()
```

Display available range of data and set frequency


```python
startDateStr = b.find_element_by_id("form:hourlyStartDate").text
endDateStr = b.find_element_by_id("form:hourlyEndDate").text
startDate = datetime.strptime(startDateStr, '%d-%m-%Y').date()
endDate = datetime.strptime(endDateStr, '%d-%m-%Y').date()
print("StartDate: %s - EndDate: %s" % (startDateStr, endDateStr))

src = b.page_source
nbdays = int(re.search(r'exceeding (?P<nbdays>\d+) days', src).group('nbdays'))
if nbdays > 365:
    r = rrule.YEARLY
elif nbdays > 30:
    r = rrule.MONTHLY
```

    StartDate: 1-1-1998 - EndDate: 31-7-2018


Let's download everything (data for all parameters)


```python
dataKeys = {'Carbon Monoxide': {'key':'co2'},
             'Nitrogen Oxides' : {'key':'nox'},
             'Sulphur Dioxide' : {'key':'so2'},
             'Fine Suspended Particulates' : {'key':'fsp'},
             'Ozone' : {'key':'o3'},
             'Nitrogen Dioxide' : {'key':'no2'},
             'Respirable Suspended Particulates' : {'key':'rsp'}}

for p in dataKeys.keys():
    dataKeys[p]['id'] = b.find_element_by_xpath('//label[contains(text(), "{0}")]'.format(p)).get_attribute('for')
    if b.find_element_by_id(dataKeys[p]['id']).get_attribute('checked') != 'true':
        b.find_element_by_id(dataKeys[p]['id']).click()
```

And finally, let's loop over all dates and download the CSV files


```python
for dt in rrule.rrule(r, dtstart=startDate, until=endDate):
    valueStart = [str(dt.year),str(dt.month),str(dt.day)]
    valueEnd = [str(dt.year),'12' if r == rrule.YEARLY else str(dt.month),'31' if r == rrule.YEARLY else str(calendar.monthrange(dt.year,dt.month)[1])]
    targetFileName = 'air_'+freq+'_'+str(dt.year)+'.csv' if r == rrule.YEARLY else 'air_'+freq+'_'+str(dt.year)+'_'+str(dt.month)+'.csv'
    if not os.path.isfile(os.path.join(endPath, targetFileName)):
        Select(b.find_element_by_id('form:dailyFromYear')).select_by_value(valueStart[0])
        Select(b.find_element_by_id('form:dailyFromMonth')).select_by_value(valueStart[1])
        Select(b.find_element_by_id('form:dailyFromDay')).select_by_value(valueStart[2])

        Select(b.find_element_by_id('form:dailyToYear')).select_by_value(valueEnd[0])
        Select(b.find_element_by_id('form:dailyToMonth')).select_by_value(valueEnd[1])
        Select(b.find_element_by_id('form:dailyToDay')).select_by_value(valueEnd[2])

        b.find_element_by_id('form:excel').click()

        while not os.path.isfile(os.path.join(dlPath, dlFileName)):
            time.sleep(1)
        print('File downloaded for %s' % str(dt))

        os.rename(os.path.join(dlPath, dlFileName), os.path.join(endPath, targetFileName))
        time.sleep(1)
    else:
        print('File already present for %s' % str(dt))
    
```

