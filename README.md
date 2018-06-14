# Exploratory Data Analysis on SMRT Tweets

This demo will provide a brief introduction in performing a rudimentary analysis on train service disruptions in Singapore. Data scrapped are from the SMRT's twitter account and wikipedia containing the relevant train stations information such as name and code 

- scraping of data from website (twitter) using Selenium 
- scraping of tabular data from website (wikipedia) using Xpath
- exploratory data analysis (EDA) on the scrapped data
- data cleaning, data prepration and processing 
- loading of .shp (shape) files into Python
- geospatial analysis on frequency of service disruptions using Folium & Leaflet

There are two primary methods of extracting data from the SMRT tweets (twitter website). The first method was to use the provided twitter API for getting SMRT tweets, while the second method was to scrap information out from the HTML codes on the official SMRT twitter website (https://twitter.com/smrt_singapore). Due to a limitation on the number of tweets the twitter's API could be pulled and an expected substantial number of SMRT tweets involved (approximately 4000 tweets), the latter method was employed to overcome twitter API's rate limitation 


## Extraction of SMRT Tweets using Selenium Web Driver

```python
import time
from selenium import webdriver
from bs4 import BeautifulSoup as bs

### Use either Firefox or Chrome to load the webpages
browser = webdriver.Firefox()
#browser = webdriver.Chrome()

### URL to scrap
url = "https://twitter.com/smrt_singapore?lang=en"

browser.get(url)
```


```python
### Selenium script to auto scroll to the end of page, with interval of 3 seconds between scroll.
### For purpose of testing, skip this part of code.

#from selenium import webdriver
#browser = webdriver.Firefox()
#browser.get("https://twitter.com/smrt_singapore?lang=en")
lenOfPage = browser.execute_script("window.scrollTo(0, document.body.scrollHeight); \
                                var lenOfPage=document.body.scrollHeight;return lenOfPage;")
match=False
while(match==False):
    lastCount = lenOfPage
    time.sleep(3)    # pause for 3 seconds before next scroll
    lenOfPage = browser.execute_script("window.scrollTo(0, document.body.scrollHeight);\
                                var lenOfPage=document.body.scrollHeight;return lenOfPage;")
    if lastCount==lenOfPage:
        match=True
```


```python
### Scrap the HTML after fully load the web page and load into BeautifulSoup.

source_data = browser.page_source
bs_data = bs(source_data, "lxml")
```


```python
### Verify data extraction for one tweet is correct.

article_info = bs_data.find("p", {"class": 'TweetTextSize TweetTextSize--normal js-tweet-text tweet-text'})
print(article_info.text)

article_info = bs_data.find('a', {'class': 'tweet-timestamp js-permalink js-nav js-tooltip' })['title']
print (article_info)
```

    All 35 EWL stns from Tuas Link to Pasir Ris & from Tanah Merah to Changi Airport will have shorter operational hrs every weekend from 6 to 29 April. Shuttle bus svcs will be available between the stns. Plan ahead & take other train lines and bus svcs. https://bit.ly/2Gvq1Br pic.twitter.com/u47bY0op8M
    2:38 AM - 2 Apr 2018



```python
### Verify REGEX pattern extracts correctly

import re

# Sample tweet
s = '[NSL] CLEARED: Train svc from #YewTee to #JurongEast is running normally now.'

repatt = r'[\bbetween\b|\bbtwn\b|\bfrom\b] \#(\w*)[\s\w]*[and|to|&]*[\s]*\#[\s]*(\w*)'
#repatt = r"\bCLEARED\b|\bcleared\b|\bCleared\b|\bhave resume\b|\bhave resumed\b|\bhas resumed\b|\bservice resumed\b|\bnormal\b|\bnormally\b|\bceased\b|\bcease\b"    # indication that fault is cleared or services is resumed
#repatt = r'\bEWL\b|\bNSL\b|\bCCL\b'
#repatt  = r'[F|f][ree][\w\s\d\&\#]*end'

extract = re.search(repatt, s)
if extract:  
    print ("From: %s  To: %s\n" % (extract.group(1), extract.group(2)))
    #print (extract.group(0))
else:
    print("Extraction Error")
```

    From: YewTee  To: JurongEast




```python
### Actual script to extract all the relevant data (tweets, date/time, Station Names, MRT Status)

from datetime import datetime
import re

article_infoAll = bs_data.find_all("div", {"class": 'content'})

smrt_tweet_data = []  # list container to hold all extracted SMRT tweet

### Regex extraction pattern
### ========================
re_mrtline = r'^[\[]*(\bNSL\b|\bEWL\b|\bCCL\b|\bBPLRT\b|\bDTL\b)'
re_mrtstatus = r'\bCLEARED\b|\bcleared\b|\bCleared\b|\bhave resume\b|\bhave resumed\b|\bhas resumed\b|\bservice resumed\b|\bnormal\b|\bnormally\b|\bceased\b|\bcease\b'    # indication that fault is cleared or services is resumed
re_patt = r'[F|f][ree][\w\s\d\&\#]*ended'
re_stn = r'[\bbetween\b|\bbtwn\b|\bfrom\b] \#(\w*)[\s\w]*[and|to|&]*[\s]*\#[\s]*(\w*)'

for item in article_infoAll:

    tweet_row = [] # list container to hold extracted tweet info for each SMRT tweet

    try:

        ### Extract tweet body from html
        ### ============================
        tweet_body = item.find("p", {"class": 'TweetTextSize TweetTextSize--normal js-tweet-text tweet-text'})
        tweet_row.append(tweet_body.text)

        ### Extract date and time from the html
        ### ===================================
        tweet_datetime = item.find('a', {'class': 'tweet-timestamp js-permalink js-nav js-tooltip' })['title']
        tweet_row.append(tweet_datetime)

        ### Convert extract tweet_datetime into datetime object so that it can be calculated
        ### =======================================
        date = datetime.strptime(tweet_datetime, '%I:%M %p - %d %b %Y')
        print("Converted date/time: %s" % date)
        tweet_row.append(date)        

        ### ================================================================================
        ### If tweet starts with [xxx], it indicate the tweet had MRT status information
        ### Extract tweet using regex to determine status type, i.e.'cleared', 'update', 'ok'
        ### ================================================================================

        ### Extract and check if tweet starts with [xxx]
        ### ============================================
        mrtline = re.search(re_mrtline, tweet_body.text)

        if mrtline:  # if the tweet starts with [XXX], then it is a status tweet
            ### Extract MRT Line inside []
            ### ==========================
            print("Extracted MRT Line: " + mrtline.group(1))
            tweet_row.append(mrtline.group(1))

            ### Extract MRT Line status based on keywords
            ### =========================================
            mrtstatus = re.search(re_mrtstatus, tweet_body.text)
            if mrtstatus:   # if is status is stated 'cleared' in tweeter
                print("Status: Cleared")
                print("Extracted tweet: %s\n" % tweet_body.text)
                tweet_row.append('cleared')
            else:
                patt=re.search(re_patt, tweet_body.text)
                if patt:    # check if there are other indicators that inferred as 'cleared'. Such as 'Free ... ended'.
                    print("Status: Cleared")
                    print("Extracted tweet: %s\n" % tweet_body.text)
                    tweet_row.append('cleared')
                else:       # if is a status tweet but no 'cleared' status, then the tweet will be a disruption status.
                    print("Status: Update")
                    print("Extracted tweet: %s" % tweet_body.text.replace('\n','')) # some tweet has newline
                    tweet_row.append('update')

                    ### Extract station names
                    ### =====================
                    stn = re.search(re_stn, tweet_body.text)
                    if extract:
                        print ("From: %s  To: %s\n" % (stn.group(1), stn.group(2)))  
                        tweet_row.append(stn.group(1))   # from station
                        tweet_row.append(stn.group(2))   # to station

        else:    # if no mrt line mention in [xxx] format, then it is a normal information tweets.
            tweet_row.append('None')
            tweet_row.append('ok')
            print('Status: OK')
            print('Extracted tweet: %s\n' % tweet_body.text)   # for checking only

        ### Store the extracted tweet info into the list container
        smrt_tweet_data.append(tweet_row)

    except:
        print("Extraction Error!\n")
        continue
```

    Converted date/time: 2018-04-02 02:38:00
    Status: OK
    Extracted tweet: All 35 EWL stns from Tuas Link to Pasir Ris & from Tanah Merah to Changi Airport will have shorter operational hrs every weekend from 6 to 29 April. Shuttle bus svcs will be available between the stns. Plan ahead & take other train lines and bus svcs. https://bit.ly/2Gvq1Br pic.twitter.com/u47bY0op8M

    Converted date/time: 2018-03-08 00:53:00
    Status: OK
    Extracted tweet: Gentle reminder that svc improvement has been made to shuttle bus svc 6 for better connectivity. Shuttle bus svc 6 will now be extended to ply from Raffles Place to Paya Lebar stns (both bounds). http://bit.ly/2oyza4d pic.twitter.com/22CPlP1JYR

    Converted date/time: 2018-03-08 00:44:00
    Status: OK
    Extracted tweet: All EWL stns from Tuas Link to Pasir Ris & from Tanah Merah to Changi Airport will have shorter operational hrs every weekend & selected weekdays from 2 Mar to 1 Apr.Shuttle bus svcs will be avail btwn the stns.Plan ahead & take other train lines/bus svcs. http://bit.ly/2FyKFkd pic.twitter.com/qj7qMVo9LJ

    Converted date/time: 2018-03-03 20:02:00
    Status: OK
    Extracted tweet: Here’s a sneak peek of our SMRT’s new uniforms, fresh from the oven! It’s been nine years since the last transformation and our staff are excited to serve you with this fresh look starting tomorrow! Look out for them at all our MRT stations and say hi!pic.twitter.com/FDi5WbZnLL





```python
### Tabulate all the extracted data into table using panda
import pandas as pd

df = pd.DataFrame(smrt_tweet_data, columns=['Tweet', 'Extracted Date/Time', 'Date/Time', 'MRT_Line', 'Status', 'From_Stn', 'To_Stn'])

df.head(10)
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>All 35 EWL stns from Tuas Link to Pasir Ris &amp; ...</td>
      <td>2:38 AM - 2 Apr 2018</td>
      <td>2018-04-02 02:38:00</td>
      <td>None</td>
      <td>ok</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Gentle reminder that svc improvement has been ...</td>
      <td>12:53 AM - 8 Mar 2018</td>
      <td>2018-03-08 00:53:00</td>
      <td>None</td>
      <td>ok</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>2</th>
      <td>All EWL stns from Tuas Link to Pasir Ris &amp; fro...</td>
      <td>12:44 AM - 8 Mar 2018</td>
      <td>2018-03-08 00:44:00</td>
      <td>None</td>
      <td>ok</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Here’s a sneak peek of our SMRT’s new uniforms...</td>
      <td>8:02 PM - 3 Mar 2018</td>
      <td>2018-03-03 20:02:00</td>
      <td>None</td>
      <td>ok</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>4</th>
      <td>All EWL stns from Tuas Link to Pasir Ris &amp; fro...</td>
      <td>3:19 AM - 1 Mar 2018</td>
      <td>2018-03-01 03:19:00</td>
      <td>None</td>
      <td>ok</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Track improvement works in the North-South Lin...</td>
      <td>3:26 AM - 19 Feb 2018</td>
      <td>2018-02-19 03:26:00</td>
      <td>None</td>
      <td>ok</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>6</th>
      <td>[NSL] CLEARED: Train svcs from #AngMoKio to #R...</td>
      <td>6:17 PM - 18 Feb 2018</td>
      <td>2018-02-18 18:17:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>7</th>
      <td>[NSL] UPDATE: Due to on going track improvemen...</td>
      <td>6:01 PM - 18 Feb 2018</td>
      <td>2018-02-18 18:01:00</td>
      <td>NSL</td>
      <td>update</td>
      <td>AngMoKio</td>
      <td>RafflesPlace</td>
    </tr>
    <tr>
      <th>8</th>
      <td>[NSL] : Due to on going track improvement work...</td>
      <td>4:29 PM - 18 Feb 2018</td>
      <td>2018-02-18 16:29:00</td>
      <td>NSL</td>
      <td>update</td>
      <td>AngMoKio</td>
      <td>RafflesPlace</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Wishing everyone a Happy and Prosperous Lunar ...</td>
      <td>8:01 AM - 15 Feb 2018</td>
      <td>2018-02-15 08:01:00</td>
      <td>None</td>
      <td>ok</td>
      <td>None</td>
      <td>None</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Number of SMRT tweets extracted
df.shape
```




    (710, 7)




```python
### Save all extracted tweets into csv
df.to_csv("smrt_tweet_extract.csv")
```


```python
### Clean up the data
### Remove 'ok' status to focus on tweet with status

df = df[df.Status != 'ok']

df.head(10)   
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>6</th>
      <td>[NSL] CLEARED: Train svcs from #AngMoKio to #R...</td>
      <td>6:17 PM - 18 Feb 2018</td>
      <td>2018-02-18 18:17:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>7</th>
      <td>[NSL] UPDATE: Due to on going track improvemen...</td>
      <td>6:01 PM - 18 Feb 2018</td>
      <td>2018-02-18 18:01:00</td>
      <td>NSL</td>
      <td>update</td>
      <td>AngMoKio</td>
      <td>RafflesPlace</td>
    </tr>
    <tr>
      <th>8</th>
      <td>[NSL] : Due to on going track improvement work...</td>
      <td>4:29 PM - 18 Feb 2018</td>
      <td>2018-02-18 16:29:00</td>
      <td>NSL</td>
      <td>update</td>
      <td>AngMoKio</td>
      <td>RafflesPlace</td>
    </tr>
    <tr>
      <th>10</th>
      <td>[NSL] CLEARED: Train svcs from #AngMoKio to #R...</td>
      <td>6:07 PM - 13 Feb 2018</td>
      <td>2018-02-13 18:07:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>11</th>
      <td>[NSL Update]: Due to maintenance work, trains ...</td>
      <td>5:04 PM - 13 Feb 2018</td>
      <td>2018-02-13 17:04:00</td>
      <td>NSL</td>
      <td>update</td>
      <td>AngMoKio</td>
      <td>Raffles</td>
    </tr>
    <tr>
      <th>12</th>
      <td>[NSL]: Due to maintenance work, south-bound tr...</td>
      <td>1:45 PM - 13 Feb 2018</td>
      <td>2018-02-13 13:45:00</td>
      <td>NSL</td>
      <td>update</td>
      <td>AngMoKio</td>
      <td>RafflesPlace</td>
    </tr>
    <tr>
      <th>13</th>
      <td>[NSL] UPDATE: Fault cleared, train svcs from #...</td>
      <td>6:38 PM - 6 Feb 2018</td>
      <td>2018-02-06 18:38:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>14</th>
      <td>[NSL] UPDATE: Due to on going track improvemen...</td>
      <td>6:04 PM - 6 Feb 2018</td>
      <td>2018-02-06 18:04:00</td>
      <td>NSL</td>
      <td>update</td>
      <td>AngMoKio</td>
      <td>RafflesPlace</td>
    </tr>
    <tr>
      <th>15</th>
      <td>[NSL] UPDATE: Fault cleared, train svcs are pr...</td>
      <td>5:18 PM - 6 Feb 2018</td>
      <td>2018-02-06 17:18:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>16</th>
      <td>[NSL] UPDATE: Fault cleared, train svcs are pr...</td>
      <td>4:46 PM - 6 Feb 2018</td>
      <td>2018-02-06 16:46:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Number of tweets with MRT Line status update
df.shape
```




    (583, 7)




```python
### Sort dataframe according to MRT line (ascending) and Date/time (descending)
### This will help in the calculation of disruption duration
df.sort_values(['MRT_Line', 'Date/Time'], ascending=[True, False], inplace=True)

df.head(10)  
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>20</th>
      <td>[BPLRT] Fault cleared. Normal train services a...</td>
      <td>12:54 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>21</th>
      <td>[BPLRT]: No train service between #ChoaChuKang...</td>
      <td>12:12 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:12:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
    </tr>
    <tr>
      <th>22</th>
      <td>[BPLRT] CLEARED: Free regular and bridging bus...</td>
      <td>3:09 AM - 12 Jan 2018</td>
      <td>2018-01-12 03:09:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>23</th>
      <td>[BPLRT] UPDATE: Train services on the entire B...</td>
      <td>2:30 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:30:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>24</th>
      <td>[BPLRT] UPDATE: Service B on the BPLRT inner l...</td>
      <td>2:04 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:04:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>165</th>
      <td>[BPLRT] CLEARED: Free bus and bridging bus ser...</td>
      <td>1:45 AM - 9 Sep 2017</td>
      <td>2017-09-09 01:45:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>166</th>
      <td>[BPLRT] CLEARED: Normal service on the BPLRT h...</td>
      <td>1:25 AM - 9 Sep 2017</td>
      <td>2017-09-09 01:25:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>203</th>
      <td>[BPLRT]\nCLEARED: Train services on the Servic...</td>
      <td>11:56 PM - 11 Aug 2017</td>
      <td>2017-08-11 23:56:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>205</th>
      <td>[BPLRT]\nCLEARED: Train services on the Servic...</td>
      <td>2:15 AM - 27 Jul 2017</td>
      <td>2017-07-27 02:15:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>206</th>
      <td>[BPLRT]\nUPDATE: Train fault between #Senja an...</td>
      <td>2:05 AM - 27 Jul 2017</td>
      <td>2017-07-27 02:05:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>Senja</td>
      <td>Jelapang</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Re-index the dataframe to prepare to calculate disruption duration
df = df.reset_index(drop=True)
df.head(10)
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[BPLRT] Fault cleared. Normal train services a...</td>
      <td>12:54 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[BPLRT]: No train service between #ChoaChuKang...</td>
      <td>12:12 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:12:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[BPLRT] CLEARED: Free regular and bridging bus...</td>
      <td>3:09 AM - 12 Jan 2018</td>
      <td>2018-01-12 03:09:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[BPLRT] UPDATE: Train services on the entire B...</td>
      <td>2:30 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:30:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[BPLRT] UPDATE: Service B on the BPLRT inner l...</td>
      <td>2:04 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:04:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>5</th>
      <td>[BPLRT] CLEARED: Free bus and bridging bus ser...</td>
      <td>1:45 AM - 9 Sep 2017</td>
      <td>2017-09-09 01:45:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>6</th>
      <td>[BPLRT] CLEARED: Normal service on the BPLRT h...</td>
      <td>1:25 AM - 9 Sep 2017</td>
      <td>2017-09-09 01:25:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>7</th>
      <td>[BPLRT]\nCLEARED: Train services on the Servic...</td>
      <td>11:56 PM - 11 Aug 2017</td>
      <td>2017-08-11 23:56:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>8</th>
      <td>[BPLRT]\nCLEARED: Train services on the Servic...</td>
      <td>2:15 AM - 27 Jul 2017</td>
      <td>2017-07-27 02:15:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>9</th>
      <td>[BPLRT]\nUPDATE: Train fault between #Senja an...</td>
      <td>2:05 AM - 27 Jul 2017</td>
      <td>2017-07-27 02:05:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>Senja</td>
      <td>Jelapang</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Calculate the disruption duration and convert into minutes.
### Taking the [Clear time] - [1st Fault Reported]
### Extract the affected train stations and placed it together with the disruption duration

cleared_found=False
curr_status=''
last_datetime=''
curr_datetime=''
cleared_datetime=''
curr_mrtlinr=''

curr_fromstn=''
curr_tostn=''
last_fromstn=''
last_tostn=''

dur_list = []
to_list=[]
fr_list=[]

for index in range(len(df)):
    curr_status = df.Status[index]
    curr_datetime = df['Date/Time'][index]

    curr_fromstn = df['From_Stn'][index]
    curr_tostn = df['To_Stn'][index]

    if index == 0:
        curr_mrtline = df.MRT_Line[index]
        last_mrtline = df.MRT_Line[index]
    else:
        curr_mrtline = df.MRT_Line[index]

    dur_list.append('0')  # default duration = 0 minutes

    to_list.append('None')
    fr_list.append('None')

    if curr_status=='cleared' and cleared_found==False:
        cleared_datetime = curr_datetime
        cleared_index = index              # remember the index so that the fault duration can be updated
        cleared_found = True

    elif cleared_found==True and (curr_status=='cleared' or curr_status=='ok'):
        #datetime_diff = cleared_datetime - last_datetime    # calculate the time difference
        datetime_diff = (cleared_datetime - last_datetime).total_seconds()/60    # calculate the time difference and convert into minutes
        print("List index position          : %s" % cleared_index)
        print("Disruption Cleared Time      : %s" % cleared_datetime)
        print("1st Disruption Reported Time : %s" % last_datetime)
        print("Disruption Duration (minutes): %s" % datetime_diff)
        print("From Station                 : %s" % last_fromstn)
        print("To Station                   : %s\n" % last_tostn)

        dur_list[cleared_index]=datetime_diff     # store the fault duration time in the list
        fr_list[cleared_index]=last_fromstn       # store the from station in the list
        to_list[cleared_index]=last_tostn         # store the to station in the list

        cleared_found=False        

        if curr_status=='cleared':
            cleared_datetime = curr_datetime
            cleared_index = index
            cleared_found = True

    last_datetime = curr_datetime
    last_mrtline = curr_mrtline

    last_fromstn = curr_fromstn
    last_tostn = curr_tostn

```

    List index position          : 0
    Disruption Cleared Time      : 2018-01-18 00:54:00
    1st Disruption Reported Time : 2018-01-18 00:12:00
    Disruption Duration (minutes): 42.0
    From Station                 : ChoaChuKang
    To Station                   : Phoenix

    List index position          : 2
    Disruption Cleared Time      : 2018-01-12 03:09:00
    1st Disruption Reported Time : 2018-01-12 03:09:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 3
    Disruption Cleared Time      : 2018-01-12 02:30:00
    1st Disruption Reported Time : 2018-01-12 02:30:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 4
    Disruption Cleared Time      : 2018-01-12 02:04:00
    1st Disruption Reported Time : 2018-01-12 02:04:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 5
    Disruption Cleared Time      : 2017-09-09 01:45:00
    1st Disruption Reported Time : 2017-09-09 01:45:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 6
    Disruption Cleared Time      : 2017-09-09 01:25:00
    1st Disruption Reported Time : 2017-09-09 01:25:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 7
    Disruption Cleared Time      : 2017-08-11 23:56:00
    1st Disruption Reported Time : 2017-08-11 23:56:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 8
    Disruption Cleared Time      : 2017-07-27 02:15:00
    1st Disruption Reported Time : 2017-07-27 02:05:00
    Disruption Duration (minutes): 10.0
    From Station                 : Senja
    To Station                   : Jelapang

    List index position          : 10
    Disruption Cleared Time      : 2017-07-19 23:03:00
    1st Disruption Reported Time : 2017-07-19 23:03:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 11
    Disruption Cleared Time      : 2017-07-19 22:58:00
    1st Disruption Reported Time : 2017-07-19 22:45:00
    Disruption Duration (minutes): 13.0
    From Station                 : ChoaChuKang
    To Station                   : BukitPanjang

    List index position          : 13
    Disruption Cleared Time      : 2017-03-28 07:02:00
    1st Disruption Reported Time : 2017-03-28 07:02:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 14
    Disruption Cleared Time      : 2017-03-28 06:28:00
    1st Disruption Reported Time : 2017-03-28 06:28:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 15
    Disruption Cleared Time      : 2017-02-10 00:12:00
    1st Disruption Reported Time : 2017-02-10 00:12:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 16
    Disruption Cleared Time      : 2017-02-09 23:43:00
    1st Disruption Reported Time : 2017-02-09 23:43:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 17
    Disruption Cleared Time      : 2016-10-26 04:37:00
    1st Disruption Reported Time : 2016-10-26 04:37:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 18
    Disruption Cleared Time      : 2016-10-26 04:32:00
    1st Disruption Reported Time : 2016-10-26 04:32:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 19
    Disruption Cleared Time      : 2016-10-26 04:12:00
    1st Disruption Reported Time : 2016-10-26 02:51:00
    Disruption Duration (minutes): 81.0
    From Station                 : BukitPanjang
    To Station                   : Phoenix

    List index position          : 21
    Disruption Cleared Time      : 2016-09-28 14:20:00
    1st Disruption Reported Time : 2016-09-28 14:20:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 22
    Disruption Cleared Time      : 2016-09-28 00:36:00
    1st Disruption Reported Time : 2016-09-28 00:36:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 23
    Disruption Cleared Time      : 2016-09-27 22:53:00
    1st Disruption Reported Time : 2016-09-27 19:56:00
    Disruption Duration (minutes): 177.0
    From Station                 : ChoaChuKang
    To Station                   : BukitPanjang

    List index position          : 26
    Disruption Cleared Time      : 2016-09-27 18:52:00
    1st Disruption Reported Time : 2016-09-27 18:52:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 27
    Disruption Cleared Time      : 2016-09-27 09:42:00
    1st Disruption Reported Time : 2016-09-27 09:42:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 28
    Disruption Cleared Time      : 2016-09-27 09:27:00
    1st Disruption Reported Time : 2016-09-27 07:55:00
    Disruption Duration (minutes): 92.0
    From Station                 : ChoaChuKang
    To Station                   : BukitPanjang

    List index position          : 30
    Disruption Cleared Time      : 2016-04-25 07:16:00
    1st Disruption Reported Time : 2016-04-25 07:16:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 31
    Disruption Cleared Time      : 2016-04-25 07:12:00
    1st Disruption Reported Time : 2016-04-25 07:12:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 32
    Disruption Cleared Time      : 2016-04-25 06:52:00
    1st Disruption Reported Time : 2016-04-25 06:52:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 33
    Disruption Cleared Time      : 2017-11-14 17:46:00
    1st Disruption Reported Time : 2017-11-14 17:46:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 34
    Disruption Cleared Time      : 2017-11-14 16:34:00
    1st Disruption Reported Time : 2017-11-14 14:28:00
    Disruption Duration (minutes): 126.00000000000001
    From Station                 : BotanicGardens
    To Station                   : HawParVilla

    List index position          : 41
    Disruption Cleared Time      : 2017-09-10 19:27:00
    1st Disruption Reported Time : 2017-09-10 18:25:00
    Disruption Duration (minutes): 62.00000000000001
    From Station                 : PayaLebar
    To Station                   : BuonaVista

    List index position          : 46
    Disruption Cleared Time      : 2017-07-21 07:28:00
    1st Disruption Reported Time : 2017-07-21 07:08:00
    Disruption Duration (minutes): 20.0
    From Station                 : Caldecott
    To Station                   : HarbourFront

    List index position          : 49
    Disruption Cleared Time      : 2017-02-23 15:43:00
    1st Disruption Reported Time : 2017-02-23 15:32:00
    Disruption Duration (minutes): 11.0
    From Station                 : bishan
    To Station                   : HarbourFront

    List index position          : 51
    Disruption Cleared Time      : 2016-11-01 20:30:00
    1st Disruption Reported Time : 2016-11-01 20:30:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 52
    Disruption Cleared Time      : 2016-11-01 20:24:00
    1st Disruption Reported Time : 2016-11-01 20:24:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 53
    Disruption Cleared Time      : 2016-11-01 20:04:00
    1st Disruption Reported Time : 2016-11-01 20:04:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 54
    Disruption Cleared Time      : 2016-11-01 19:46:00
    1st Disruption Reported Time : 2016-11-01 17:09:00
    Disruption Duration (minutes): 157.0
    From Station                 : BotanicGardens
    To Station                   : HarbourFront

    List index position          : 63
    Disruption Cleared Time      : 2016-11-01 16:39:00
    1st Disruption Reported Time : 2016-11-01 16:37:00
    Disruption Duration (minutes): 2.0000000000000004
    From Station                 : PasirPanjang
    To Station                   : one

    List index position          : 65
    Disruption Cleared Time      : 2016-09-19 19:15:00
    1st Disruption Reported Time : 2016-09-19 19:15:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 66
    Disruption Cleared Time      : 2016-09-19 19:03:00
    1st Disruption Reported Time : 2016-09-19 16:13:00
    Disruption Duration (minutes): 170.0
    From Station                 : DhobyGhaut
    To Station                   : MacPherson

    List index position          : 74
    Disruption Cleared Time      : 2016-09-19 16:06:00
    1st Disruption Reported Time : 2016-09-19 15:57:00
    Disruption Duration (minutes): 9.0
    From Station                 : PayaLebar
    To Station                   : MacPherson

    List index position          : 76
    Disruption Cleared Time      : 2016-09-05 06:13:00
    1st Disruption Reported Time : 2016-09-05 05:22:00
    Disruption Duration (minutes): 51.0
    From Station                 : PayaLebar
    To Station                   : MacPherson

    List index position          : 80
    Disruption Cleared Time      : 2016-08-31 23:59:00
    1st Disruption Reported Time : 2016-08-31 23:59:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 81
    Disruption Cleared Time      : 2016-08-30 21:44:00
    1st Disruption Reported Time : 2016-08-30 21:44:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 82
    Disruption Cleared Time      : 2016-04-25 06:31:00
    1st Disruption Reported Time : 2016-04-25 06:31:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 83
    Disruption Cleared Time      : 2016-04-25 06:23:00
    1st Disruption Reported Time : 2016-04-25 06:23:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 84
    Disruption Cleared Time      : 2018-01-10 14:43:00
    1st Disruption Reported Time : 2018-01-10 14:43:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 85
    Disruption Cleared Time      : 2018-01-10 14:33:00
    1st Disruption Reported Time : 2018-01-10 14:19:00
    Disruption Duration (minutes): 14.0
    From Station                 : OutramPark
    To Station                   : Bugis

    List index position          : 87
    Disruption Cleared Time      : 2018-01-01 17:02:00
    1st Disruption Reported Time : 2018-01-01 17:02:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 88
    Disruption Cleared Time      : 2018-01-01 16:59:00
    1st Disruption Reported Time : 2018-01-01 15:53:00
    Disruption Duration (minutes): 66.00000000000001
    From Station                 : TanahMerah
    To Station                   : ChangiAirport

    List index position          : 91
    Disruption Cleared Time      : 2018-01-01 15:17:00
    1st Disruption Reported Time : 2018-01-01 13:49:00
    Disruption Duration (minutes): 88.0
    From Station                 : ChangiAirport
    To Station                   : TanahMerah

    List index position          : 96
    Disruption Cleared Time      : 2017-12-06 14:24:00
    1st Disruption Reported Time : 2017-12-06 14:24:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 97
    Disruption Cleared Time      : 2017-12-06 14:05:00
    1st Disruption Reported Time : 2017-11-15 04:29:00
    Disruption Duration (minutes): 30816.0
    From Station                 : JooKoon
    To Station                   : TuasLink

    List index position          : 106
    Disruption Cleared Time      : 2017-11-15 01:34:00
    1st Disruption Reported Time : 2017-11-15 01:34:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 107
    Disruption Cleared Time      : 2017-11-15 01:15:00
    1st Disruption Reported Time : 2017-11-10 07:15:00
    Disruption Duration (minutes): 6840.0
    From Station                 : TiongBahru
    To Station                   : PasirRis

    List index position          : 120
    Disruption Cleared Time      : 2017-11-10 06:58:00
    1st Disruption Reported Time : 2017-11-10 06:58:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 121
    Disruption Cleared Time      : 2017-11-10 06:37:00
    1st Disruption Reported Time : 2017-11-10 05:53:00
    Disruption Duration (minutes): 44.0
    From Station                 : Bugis
    To Station                   : Queenstown

    List index position          : 124
    Disruption Cleared Time      : 2017-11-04 00:34:00
    1st Disruption Reported Time : 2017-11-04 00:34:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 125
    Disruption Cleared Time      : 2017-11-04 00:30:00
    1st Disruption Reported Time : 2017-11-03 23:53:00
    Disruption Duration (minutes): 37.0
    From Station                 : Queenstown
    To Station                   : JurongEast

    List index position          : 129
    Disruption Cleared Time      : 2017-09-27 17:37:00
    1st Disruption Reported Time : 2017-09-27 16:15:00
    Disruption Duration (minutes): 82.0
    From Station                 : Tampines
    To Station                   : PasirRis

    List index position          : 132
    Disruption Cleared Time      : 2017-09-27 15:47:00
    1st Disruption Reported Time : 2017-09-27 15:47:00
    Disruption Duration (minutes): 0.0
    From Station                 : None
    To Station                   : None

    List index position          : 133
    Disruption Cleared Time      : 2017-09-27 15:15:00
    1st Disruption Reported Time : 2017-09-27 14:54:00
    Disruption Duration (minutes): 21.0
    From Station                 : TanahMerah
    To Station                   : PasirRis






```python
### Combine the duration, from and to list into existing df panda dataframe
df['Duration'] = dur_list
df['New_Fr_Stn'] = fr_list
df['New_To_Stn'] = to_list

df.head(70)
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
      <th>Duration</th>
      <th>New_Fr_Stn</th>
      <th>New_To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[BPLRT] Fault cleared. Normal train services a...</td>
      <td>12:54 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>42</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[BPLRT]: No train service between #ChoaChuKang...</td>
      <td>12:12 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:12:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[BPLRT] CLEARED: Free regular and bridging bus...</td>
      <td>3:09 AM - 12 Jan 2018</td>
      <td>2018-01-12 03:09:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[BPLRT] UPDATE: Train services on the entire B...</td>
      <td>2:30 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:30:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[BPLRT] UPDATE: Service B on the BPLRT inner l...</td>
      <td>2:04 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:04:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>5</th>
      <td>[BPLRT] CLEARED: Free bus and bridging bus ser...</td>
      <td>1:45 AM - 9 Sep 2017</td>
      <td>2017-09-09 01:45:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>6</th>
      <td>[BPLRT] CLEARED: Normal service on the BPLRT h...</td>
      <td>1:25 AM - 9 Sep 2017</td>
      <td>2017-09-09 01:25:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>7</th>
      <td>[BPLRT]\nCLEARED: Train services on the Servic...</td>
      <td>11:56 PM - 11 Aug 2017</td>
      <td>2017-08-11 23:56:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>8</th>
      <td>[BPLRT]\nCLEARED: Train services on the Servic...</td>
      <td>2:15 AM - 27 Jul 2017</td>
      <td>2017-07-27 02:15:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>10</td>
      <td>Senja</td>
      <td>Jelapang</td>
    </tr>
    <tr>
      <th>9</th>
      <td>[BPLRT]\nUPDATE: Train fault between #Senja an...</td>
      <td>2:05 AM - 27 Jul 2017</td>
      <td>2017-07-27 02:05:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>Senja</td>
      <td>Jelapang</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>10</th>
      <td>[BPLRT] CLEARED: Normal service resumed. Free ...</td>
      <td>11:03 PM - 19 Jul 2017</td>
      <td>2017-07-19 23:03:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>11</th>
      <td>[BPLRT] CLEARED: Train service between #ChoaCh...</td>
      <td>10:58 PM - 19 Jul 2017</td>
      <td>2017-07-19 22:58:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>13</td>
      <td>ChoaChuKang</td>
      <td>BukitPanjang</td>
    </tr>
    <tr>
      <th>12</th>
      <td>[BPLRT] No train service betwn #ChoaChuKang to...</td>
      <td>10:45 PM - 19 Jul 2017</td>
      <td>2017-07-19 22:45:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>BukitPanjang</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>13</th>
      <td>[BPLRT] Update: Train services have resumed. B...</td>
      <td>7:02 AM - 28 Mar 2017</td>
      <td>2017-03-28 07:02:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>14</th>
      <td>[BPLRT] Update: Train services have resumed. B...</td>
      <td>6:28 AM - 28 Mar 2017</td>
      <td>2017-03-28 06:28:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>15</th>
      <td>[BPLRT CLEARED] Free Regular and Shuttle Bus S...</td>
      <td>12:12 AM - 10 Feb 2017</td>
      <td>2017-02-10 00:12:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>16</th>
      <td>[BPLRT Update] Normal Train Services have resu...</td>
      <td>11:43 PM - 9 Feb 2017</td>
      <td>2017-02-09 23:43:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>17</th>
      <td>[BPLRT] UPDATE: Train services have resumed. F...</td>
      <td>4:37 AM - 26 Oct 2016</td>
      <td>2016-10-26 04:37:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>18</th>
      <td>[BPLRT] UPDATE: Train services have resumed. F...</td>
      <td>4:32 AM - 26 Oct 2016</td>
      <td>2016-10-26 04:32:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>19</th>
      <td>[BPLRT] UPDATE: Train services have resumed. F...</td>
      <td>4:12 AM - 26 Oct 2016</td>
      <td>2016-10-26 04:12:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>81</td>
      <td>BukitPanjang</td>
      <td>Phoenix</td>
    </tr>
    <tr>
      <th>20</th>
      <td>[BPLRT] Due to a train fault between #BukitPan...</td>
      <td>2:51 AM - 26 Oct 2016</td>
      <td>2016-10-26 02:51:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>BukitPanjang</td>
      <td>Phoenix</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>21</th>
      <td>[BPLRT] Service A and Service B are now runnin...</td>
      <td>2:20 PM - 28 Sep 2016</td>
      <td>2016-09-28 14:20:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>22</th>
      <td>[BPLRT]Svcs on the inner loop (anticlockwise d...</td>
      <td>12:36 AM - 28 Sep 2016</td>
      <td>2016-09-28 00:36:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>23</th>
      <td>[BPLRT] Svcs on the inner loop (anticlockwise ...</td>
      <td>10:53 PM - 27 Sep 2016</td>
      <td>2016-09-27 22:53:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>177</td>
      <td>ChoaChuKang</td>
      <td>BukitPanjang</td>
    </tr>
    <tr>
      <th>24</th>
      <td>[BPLRT] No train service btwn #ChoaChuKang and...</td>
      <td>8:25 PM - 27 Sep 2016</td>
      <td>2016-09-27 20:25:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>BukitPanjang</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>25</th>
      <td>[BPLRT] No train service between #ChoaChuKang ...</td>
      <td>7:56 PM - 27 Sep 2016</td>
      <td>2016-09-27 19:56:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>BukitPanjang</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>26</th>
      <td>[BPLRT Update] Free regular bus &amp; free bridgin...</td>
      <td>6:52 PM - 27 Sep 2016</td>
      <td>2016-09-27 18:52:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>27</th>
      <td>[BPLRT] CLEARED: Free regular bus &amp; free bridg...</td>
      <td>9:42 AM - 27 Sep 2016</td>
      <td>2016-09-27 09:42:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>28</th>
      <td>[BPLRT] CLEARED: Train service between #ChoaCh...</td>
      <td>9:27 AM - 27 Sep 2016</td>
      <td>2016-09-27 09:27:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>92</td>
      <td>ChoaChuKang</td>
      <td>BukitPanjang</td>
    </tr>
    <tr>
      <th>29</th>
      <td>[BPLRT] No train service between #ChoaChuKang ...</td>
      <td>7:55 AM - 27 Sep 2016</td>
      <td>2016-09-27 07:55:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>BukitPanjang</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>40</th>
      <td>[CCL]: Due to a signal  fault trains are movin...</td>
      <td>2:28 PM - 14 Nov 2017</td>
      <td>2017-11-14 14:28:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>BotanicGardens</td>
      <td>HawParVilla</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>41</th>
      <td>[CCL] Update: Train services are running norma...</td>
      <td>7:27 PM - 10 Sep 2017</td>
      <td>2017-09-10 19:27:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>62</td>
      <td>PayaLebar</td>
      <td>BuonaVista</td>
    </tr>
    <tr>
      <th>42</th>
      <td>[CCL Update] Pls add 30mins additional travell...</td>
      <td>6:54 PM - 10 Sep 2017</td>
      <td>2017-09-10 18:54:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>PayaLebar</td>
      <td>BuonaVista</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>43</th>
      <td>[CCL Update] Pls add 30mins additional travell...</td>
      <td>6:40 PM - 10 Sep 2017</td>
      <td>2017-09-10 18:40:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>PayaLebar</td>
      <td>BuonaVista</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>44</th>
      <td>[CCL] UPDATE: Free regular bus services availa...</td>
      <td>6:27 PM - 10 Sep 2017</td>
      <td>2017-09-10 18:27:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>PayaLebar</td>
      <td>BuonaVista</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>45</th>
      <td>[CCL] Pls add 15mins additional travelling tim...</td>
      <td>6:25 PM - 10 Sep 2017</td>
      <td>2017-09-10 18:25:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>PayaLebar</td>
      <td>BuonaVista</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>46</th>
      <td>[CCL] UPDATE: Normal train service resumed tow...</td>
      <td>7:28 AM - 21 Jul 2017</td>
      <td>2017-07-21 07:28:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>20</td>
      <td>Caldecott</td>
      <td>HarbourFront</td>
    </tr>
    <tr>
      <th>47</th>
      <td>[CCL] UPDATE: Free regular bus services are av...</td>
      <td>7:09 AM - 21 Jul 2017</td>
      <td>2017-07-21 07:09:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>Caldecott</td>
      <td>HarbourFront</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>48</th>
      <td>[CCL]: Estimate 15mins additional travelling t...</td>
      <td>7:08 AM - 21 Jul 2017</td>
      <td>2017-07-21 07:08:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>Caldecott</td>
      <td>HarbourFront</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>49</th>
      <td>CCL: CLEARED: Train Services between #Bishan a...</td>
      <td>3:43 PM - 23 Feb 2017</td>
      <td>2017-02-23 15:43:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>11</td>
      <td>bishan</td>
      <td>HarbourFront</td>
    </tr>
    <tr>
      <th>50</th>
      <td>CCL: Estimate 5 mins additional travelling tim...</td>
      <td>3:32 PM - 23 Feb 2017</td>
      <td>2017-02-23 15:32:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>bishan</td>
      <td>HarbourFront</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>51</th>
      <td>[CCL] Bus bridging and free regular bus svcs h...</td>
      <td>8:30 PM - 1 Nov 2016</td>
      <td>2016-11-01 20:30:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>52</th>
      <td>[CCL] Train svcs have resumed. Bus bridging sv...</td>
      <td>8:24 PM - 1 Nov 2016</td>
      <td>2016-11-01 20:24:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>53</th>
      <td>[CCL] Train svcs have resumed. Bus bridging sv...</td>
      <td>8:04 PM - 1 Nov 2016</td>
      <td>2016-11-01 20:04:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>54</th>
      <td>[CCL] Train svcs have resumed. Bus bridging sv...</td>
      <td>7:46 PM - 1 Nov 2016</td>
      <td>2016-11-01 19:46:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>157</td>
      <td>BotanicGardens</td>
      <td>HarbourFront</td>
    </tr>
    <tr>
      <th>55</th>
      <td>[CCL] Train svcs just resumed. Bus bridging sv...</td>
      <td>7:27 PM - 1 Nov 2016</td>
      <td>2016-11-01 19:27:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>Bishan</td>
      <td>PayaLebar</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>56</th>
      <td>[CCL]Bus bridging svcs avail between #Bishan a...</td>
      <td>7:20 PM - 1 Nov 2016</td>
      <td>2016-11-01 19:20:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>Bishan</td>
      <td>PayaLebar</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>57</th>
      <td>[CCL} Bus bridging svcs avail between #Bishan ...</td>
      <td>7:01 PM - 1 Nov 2016</td>
      <td>2016-11-01 19:01:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>Bishan</td>
      <td>PayaLebar</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>58</th>
      <td>[CCL]No train svc btwn #BotanicGardens and #Se...</td>
      <td>6:50 PM - 1 Nov 2016</td>
      <td>2016-11-01 18:50:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>BotanicGardens</td>
      <td>Serangoon</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>59</th>
      <td>[CCL] No train svc btwn #BotanicGardens and #S...</td>
      <td>6:32 PM - 1 Nov 2016</td>
      <td>2016-11-01 18:32:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>BotanicGardens</td>
      <td>Serangoon</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>60</th>
      <td>[CCL] No train svc between #BotanicGardens and...</td>
      <td>6:26 PM - 1 Nov 2016</td>
      <td>2016-11-01 18:26:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>BotanicGardens</td>
      <td>Marymount</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>61</th>
      <td>[CCL] No train service between #BotanicGardens...</td>
      <td>6:06 PM - 1 Nov 2016</td>
      <td>2016-11-01 18:06:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>BotanicGardens</td>
      <td>Marymount</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>62</th>
      <td>[CCL] UPDATE: Estimate 10 mins additional trav...</td>
      <td>5:09 PM - 1 Nov 2016</td>
      <td>2016-11-01 17:09:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>BotanicGardens</td>
      <td>HarbourFront</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>63</th>
      <td>[CCL] CLEARED: Fault cleared but trains and st...</td>
      <td>4:39 PM - 1 Nov 2016</td>
      <td>2016-11-01 16:39:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>2</td>
      <td>PasirPanjang</td>
      <td>one</td>
    </tr>
    <tr>
      <th>64</th>
      <td>[CCL]\n  UPDATE: Estimate 10 mins additional t...</td>
      <td>4:37 PM - 1 Nov 2016</td>
      <td>2016-11-01 16:37:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>PasirPanjang</td>
      <td>one</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>65</th>
      <td>[CCL] CLEARED: Train service is running normal...</td>
      <td>7:15 PM - 19 Sep 2016</td>
      <td>2016-09-19 19:15:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>66</th>
      <td>[CCL] Update: Free regular bus service has cea...</td>
      <td>7:03 PM - 19 Sep 2016</td>
      <td>2016-09-19 19:03:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>170</td>
      <td>DhobyGhaut</td>
      <td>MacPherson</td>
    </tr>
    <tr>
      <th>67</th>
      <td>[CCL] Update: Estimate 10mins additional trave...</td>
      <td>6:53 PM - 19 Sep 2016</td>
      <td>2016-09-19 18:53:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>DhobyGhaut</td>
      <td>Bishan</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>68</th>
      <td>[CCL] Update: Estimate 15 mins additional trav...</td>
      <td>6:41 PM - 19 Sep 2016</td>
      <td>2016-09-19 18:41:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>DhobyGhaut</td>
      <td>Bishan</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>69</th>
      <td>[CCL] Update: Free regular bus services are av...</td>
      <td>5:49 PM - 19 Sep 2016</td>
      <td>2016-09-19 17:49:00</td>
      <td>CCL</td>
      <td>update</td>
      <td>PayaLebar</td>
      <td>Bishan</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
  </tbody>
</table>
<p>70 rows × 10 columns</p>
</div>




## Checking for outliers


```python
### Check for outliers, i.e see any adnormal spikes that may indicate data error
import matplotlib.pyplot as plt
%matplotlib inline
import seaborn

df = df.sort_values('Date/Time', ascending=True)
plt.plot(df['Date/Time'], df['Duration'])
plt.xticks(rotation='vertical')
plt.show()
```


![png](images/output_17_0.png)



```python
### Another view to check for outliers details that has duration > 1 day (1440 durations),
### which may indicate data error

count=0
outlier_list=[]
for index in range(len(df)):
    #if df.Duration[index].days >=  1:
    if int(df.Duration[index]) >=  1440:  # 1440 = 24 hours
        print ('Disruption >1 day (1440 minutes) at index %s: %s, %s, %s' % (index, df.Duration[index], df['Date/Time'][index], df.MRT_Line[index]))
        count = count + 1
        outlier_list.append(index)  # store the outlier index

print("Count of disruption >1 day (1440 minutes): %s" % count)
print(outlier_list)

### Sort the outlier index
outlier_list.sort()
outlier_list=outlier_list[::-1]
print(outlier_list)
```

    Disruption >1 day (1440 minutes) at index 97: 30816.0, 2017-12-06 14:05:00, EWL
    Disruption >1 day (1440 minutes) at index 107: 6840.0, 2017-11-15 01:15:00, EWL
    Disruption >1 day (1440 minutes) at index 403: 11615.0, 2017-08-31 03:55:00, NSL
    Count of disruption >1 day (1440 minutes): 3
    [97, 107, 403]
    [403, 107, 97]



```python
### Remove outliers from the dataframe after investigating the data.

for i in outlier_list:
    print("Removing data at index %s." % i)
    df = df[df.index!= i]
```

    Removing data at index 403.
    Removing data at index 107.
    Removing data at index 97.



```python
### Check if outlier rows has been deleted
df.shape
```




    (580, 10)




```python
### Re-Sort the MRT Line and Date/time after removing outliers
df.sort_values(['MRT_Line', 'Date/Time'], ascending=[True, False], inplace=True)
df.head()
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
      <th>Duration</th>
      <th>New_Fr_Stn</th>
      <th>New_To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[BPLRT] Fault cleared. Normal train services a...</td>
      <td>12:54 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>42</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[BPLRT]: No train service between #ChoaChuKang...</td>
      <td>12:12 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:12:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[BPLRT] CLEARED: Free regular and bridging bus...</td>
      <td>3:09 AM - 12 Jan 2018</td>
      <td>2018-01-12 03:09:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[BPLRT] UPDATE: Train services on the entire B...</td>
      <td>2:30 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:30:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[BPLRT] UPDATE: Service B on the BPLRT inner l...</td>
      <td>2:04 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:04:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
  </tbody>
</table>
</div>


## EDA on average service disruption (mins)

```python
### Compute the average disruption duration
count=0
total=0
for index in range(len(df)):
    if index in outlier_list: pass
    elif int(df.Duration[index]) > 0:
        total = total + int(df.Duration[index])
        count = count + 1
        datetime = df['Date/Time'][index]
        #print(index)

average = total/count

print('Number of disruption since %s                 : %s' % (datetime, count))
print("Average SMRT Line disruption duration since %s: %.2f minutes" % (datetime, average))
```

    Number of disruption since 2016-05-11 01:41:00                 : 114
    Average SMRT Line disruption duration since 2016-05-11 01:41:00: 75.33 minutes



```python
### Plot disruption duration (in minutes) against date/time with mean

df = df.sort_values('Date/Time', ascending=True)

y_mean = [average for i in df['Date/Time']]

plt.plot(df['Date/Time'], df['Duration'])
plt.plot(df['Date/Time'], y_mean, label='Mean', linestyle='--')

legend = plt.legend(loc='best')  # insert a legend

plt.show()
```


![png](images/output_23_0.png)


## Analysing relationship between number of tweets and disruption occurances 

```python
### Prepare dataset for analysing relationship between number of tweets and disruption
### ==================================================================================

### Remove all non-'cleared' status

df2 = df[df.Status != 'update']
df2.head(10)   
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
      <th>Duration</th>
      <th>New_Fr_Stn</th>
      <th>New_To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>83</th>
      <td>[CCL] Normal services have resumed. Free shutt...</td>
      <td>6:23 AM - 25 Apr 2016</td>
      <td>2016-04-25 06:23:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>82</th>
      <td>[CCL] Normal services have resumed. Free shutt...</td>
      <td>6:31 AM - 25 Apr 2016</td>
      <td>2016-04-25 06:31:00</td>
      <td>CCL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>32</th>
      <td>[BPLRT] Train services have resumed on the BPL...</td>
      <td>6:52 AM - 25 Apr 2016</td>
      <td>2016-04-25 06:52:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>31</th>
      <td>[BPLRT] Train services have resumed on BPLRT. ...</td>
      <td>7:12 AM - 25 Apr 2016</td>
      <td>2016-04-25 07:12:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>30</th>
      <td>[BPLRT] Free shuttle bus services have ceased.</td>
      <td>7:16 AM - 25 Apr 2016</td>
      <td>2016-04-25 07:16:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>580</th>
      <td>[NSL] CLEARED: Train service from #Kranji to #...</td>
      <td>3:49 PM - 25 Apr 2016</td>
      <td>2016-04-25 15:49:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>579</th>
      <td>[NSL]CLEARED: Free public &amp; shuttle buses are ...</td>
      <td>3:51 PM - 25 Apr 2016</td>
      <td>2016-04-25 15:51:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>578</th>
      <td>[NSL]Fault cleared but some trains &amp; stations ...</td>
      <td>4:20 PM - 25 Apr 2016</td>
      <td>2016-04-25 16:20:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>577</th>
      <td>[NSL] CLEARED: Train services between Kranji a...</td>
      <td>4:29 PM - 25 Apr 2016</td>
      <td>2016-04-25 16:29:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>333</th>
      <td>[EWL] CLEARED: Train service has resumed betwe...</td>
      <td>1:33 AM - 2 May 2016</td>
      <td>2016-05-02 01:33:00</td>
      <td>EWL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>49</td>
      <td>JooKoon</td>
      <td>BoonLay</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Check data type

df['Duration'] = df['Duration'].astype(int)
df.dtypes
```




    Tweet                          object
    Extracted Date/Time            object
    Date/Time              datetime64[ns]
    MRT_Line                       object
    Status                         object
    From_Stn                       object
    To_Stn                         object
    Duration                        int64
    New_Fr_Stn                     object
    New_To_Stn                     object
    dtype: object




```python
### Remove duration=0 and put the clean up data into a new dataframe
df2 = df[df.Duration != 0]
df2.head(10)
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
      <th>Duration</th>
      <th>New_Fr_Stn</th>
      <th>New_To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>333</th>
      <td>[EWL] CLEARED: Train service has resumed betwe...</td>
      <td>1:33 AM - 2 May 2016</td>
      <td>2016-05-02 01:33:00</td>
      <td>EWL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>49</td>
      <td>JooKoon</td>
      <td>BoonLay</td>
    </tr>
    <tr>
      <th>331</th>
      <td>[EWL] CLEARED: Free regular bus service betwee...</td>
      <td>1:53 AM - 2 May 2016</td>
      <td>2016-05-02 01:53:00</td>
      <td>EWL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>16</td>
      <td>JooKoon</td>
      <td>BoonLay</td>
    </tr>
    <tr>
      <th>572</th>
      <td>[NSL] CLEARED: Train services from #Yishun tow...</td>
      <td>1:41 AM - 11 May 2016</td>
      <td>2016-05-11 01:41:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>40</td>
      <td>Yishun</td>
      <td>Yio</td>
    </tr>
    <tr>
      <th>569</th>
      <td>[NSL]UPDATE:Train service from Woodlands to Se...</td>
      <td>1:53 AM - 23 Jun 2016</td>
      <td>2016-06-23 01:53:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>8</td>
      <td>Woodlands</td>
      <td>Sembawang</td>
    </tr>
    <tr>
      <th>563</th>
      <td>[NSL] CLEARED: Normal train service from #YioC...</td>
      <td>7:49 AM - 29 Jun 2016</td>
      <td>2016-06-29 07:49:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>90</td>
      <td>Woodlands</td>
      <td>AngMoKio</td>
    </tr>
    <tr>
      <th>561</th>
      <td>[NSL] UPDATE: South Bound normal service resum...</td>
      <td>8:22 PM - 12 Jul 2016</td>
      <td>2016-07-12 20:22:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>10</td>
      <td>Bishan</td>
      <td>Khatib</td>
    </tr>
    <tr>
      <th>559</th>
      <td>[NSL] CLEARED: Normal train service from #Bish...</td>
      <td>9:07 PM - 12 Jul 2016</td>
      <td>2016-07-12 21:07:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>30</td>
      <td>Bishan</td>
      <td>Khatib</td>
    </tr>
    <tr>
      <th>555</th>
      <td>[NSL] CLEARED: Train service between #MarinaBa...</td>
      <td>5:02 PM - 20 Jul 2016</td>
      <td>2016-07-20 17:02:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>18</td>
      <td>MarinaBay</td>
      <td>MarinaSouthPier</td>
    </tr>
    <tr>
      <th>327</th>
      <td>[EWL] Update: Fault has been cleared. However,...</td>
      <td>2:48 AM - 31 Jul 2016</td>
      <td>2016-07-31 02:48:00</td>
      <td>EWL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>54</td>
      <td>Bugis</td>
      <td>PasirRis</td>
    </tr>
    <tr>
      <th>550</th>
      <td>[NSL] CLEARED: Train service on the NSL has re...</td>
      <td>6:19 PM - 1 Aug 2016</td>
      <td>2016-08-01 18:19:00</td>
      <td>NSL</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>21</td>
      <td>Yishun</td>
      <td>Woodlands</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Plot the relationship between tweets and disruption data
fig = plt.figure()
ax = plt.subplot(111)

df.groupby('MRT_Line').Status.count().plot(kind='bar', ax=ax, label='Tweets', color='g')
df2.groupby('MRT_Line').Status.count().plot(kind='bar', ax=ax, label='Disruption', color='b')

legend = plt.legend(loc='best')
```


![png](images/output_27_0.png)


## EDA on disruption occurances 

```python
### Prepare a new dataframe for disruption analysis
df3 = pd.DataFrame()
df3['MRT_Line'] = df2['MRT_Line']
df3['hours'] = df2['Date/Time'].dt.hour
df3['duration'] = df2['Duration']
df3['Date/Time'] = df2['Date/Time']
df3.head(10)
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>MRT_Line</th>
      <th>hours</th>
      <th>duration</th>
      <th>Date/Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>333</th>
      <td>EWL</td>
      <td>1</td>
      <td>49</td>
      <td>2016-05-02 01:33:00</td>
    </tr>
    <tr>
      <th>331</th>
      <td>EWL</td>
      <td>1</td>
      <td>16</td>
      <td>2016-05-02 01:53:00</td>
    </tr>
    <tr>
      <th>572</th>
      <td>NSL</td>
      <td>1</td>
      <td>40</td>
      <td>2016-05-11 01:41:00</td>
    </tr>
    <tr>
      <th>569</th>
      <td>NSL</td>
      <td>1</td>
      <td>8</td>
      <td>2016-06-23 01:53:00</td>
    </tr>
    <tr>
      <th>563</th>
      <td>NSL</td>
      <td>7</td>
      <td>90</td>
      <td>2016-06-29 07:49:00</td>
    </tr>
    <tr>
      <th>561</th>
      <td>NSL</td>
      <td>20</td>
      <td>10</td>
      <td>2016-07-12 20:22:00</td>
    </tr>
    <tr>
      <th>559</th>
      <td>NSL</td>
      <td>21</td>
      <td>30</td>
      <td>2016-07-12 21:07:00</td>
    </tr>
    <tr>
      <th>555</th>
      <td>NSL</td>
      <td>17</td>
      <td>18</td>
      <td>2016-07-20 17:02:00</td>
    </tr>
    <tr>
      <th>327</th>
      <td>EWL</td>
      <td>2</td>
      <td>54</td>
      <td>2016-07-31 02:48:00</td>
    </tr>
    <tr>
      <th>550</th>
      <td>NSL</td>
      <td>18</td>
      <td>21</td>
      <td>2016-08-01 18:19:00</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Analyse the MRT service breakdown ocurrence

from matplotlib.ticker import FormatStrFormatter
import seaborn as sns

xFormatter = FormatStrFormatter('%02d:00')

snsplot = sns.lmplot('hours', 'duration', data=df3, hue='MRT_Line', fit_reg=False, size=7) #remove col='MRT_Line' to get a combine view,
snsplot.set(xlabel = "Time (24hrs)", ylabel = "Disruption Duration (mins)")#, title = "Disruption Plot")

axes = snsplot.ax
axes.xaxis.set_major_formatter(xFormatter)

plt.show()
```


![png](images/output_29_0.png)



```python
### Plot disruption occurence chart for each MRT Line separately

xFormatter = FormatStrFormatter('%02d:00')

snsplot = sns.lmplot('hours', 'duration', data=df3, hue='MRT_Line', col='MRT_Line', col_wrap=2, fit_reg=False, size=5) #remove col='MRT_Line' to get a combine view
#sharex=True, sharey=True,
snsplot.set(xlabel = "Time (24hrs)", ylabel = "Disruption Duration (mins)")#, title = "Disruption Plot")

plt.show()

```


![png](images/output_30_0.png)



```python
### Frequency of service disruption across time (hourly) of day
fig, ax = plt.subplots()
df3.groupby('hours').MRT_Line.count().plot.bar()
ax.set_xlabel("Time of Day")
```




    <matplotlib.text.Text at 0x1286df908>




![png](images/output_31_1.png)



```python
### Yearly frequency of service disruption
fig, ax = plt.subplots()
df3.groupby(df3['Date/Time'].dt.year).MRT_Line.count().plot.bar()
ax.set_xlabel("Year")
```




    <matplotlib.text.Text at 0x1289af7f0>




![png](images/output_32_1.png)



```python
### Monthly frequency of service disruption

mths=['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec']

fig, ax = plt.subplots()
df3.groupby(df3['Date/Time'].dt.month).MRT_Line.count().plot.bar()
ax.set_xlabel("Month")
ax.set_xticklabels([mths[i] for i in range(12)])
```



![png](images/output_33_1.png)



```python
### Day of week daily frequency of service disruption

days=['Sun','Mon','Tue','Wed','Thu','Fri','Sat']

fig, ax = plt.subplots()
df3.groupby(df3['Date/Time'].dt.dayofweek).MRT_Line.count().plot.bar()

ax.set_xlabel("Day")
ax.set_xticklabels([days[i] for i in range(7)])
```





![png](images/output_34_1.png)



```python
browser.quit()
```

## sdasda

```python
import matplotlib.pyplot as plt
%matplotlib inline
#matplotlib.rcParams['figure.figsize'] = [14, 8]
import requests
from lxml import html
import pandas as pd 
import re

#################################################
# Scraping of related MRT/LRT stations details #
################################################

removeWords = set(['\n', 'Reserved station', 'N/A', 'Reserved Station', '[a]'])
```


```python
## Scraping North South Line stations from wiki page using xPath
allStn=requests.get('https://en.wikipedia.org/wiki/List_of_Singapore_MRT_stations')
allStnTree = html.fromstring(allStn.content)
allStnName = allStnTree.xpath('//table[@class="wikitable"]//td[2]//text()')
allStnName.pop(allStnName.index('[a]'))
allStnName.pop(allStnName.index('Canberra'))

startingXpath = '//table[@class="wikitable"]//tr[3]//text()'
startingXpath = int(re.search(r'[0-9]+', startingXpath)[0])

endingXpath = '//table[@class="wikitable"]//tr[32]//text()'
endingXpath = int(re.search(r'[0-9]+', endingXpath)[0])

redEndPos = endingXpath - startingXpath - 1

redStnName = allStnName[:redEndPos]
redStnName = list(filter(lambda x: x not in removeWords, redStnName))

nsl_df = pd.DataFrame(redStnName, columns=['Name'])
nsl_df['Line'] = 'NSL'
nsl_df.head()
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
      <th>Name</th>
      <th>Line</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Jurong East</td>
      <td>NSL</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Bukit Batok</td>
      <td>NSL</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Bukit Gombak</td>
      <td>NSL</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Choa Chu Kang</td>
      <td>NSL</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Yew Tee</td>
      <td>NSL</td>
    </tr>
  </tbody>
</table>
</div>




```python
## Scraping East West Line stations from wiki page using xPath
startingXpath = '//table[@class="wikitable"]//tr[32]//text()'
startingXpath = int(re.search(r'[0-9]+', startingXpath)[0])

endingXpath = '//table[@class="wikitable"]//tr[66]//text()'
endingXpath = int(re.search(r'[0-9]+', endingXpath)[0])

greenPos = endingXpath - startingXpath - 1
greenEndPos = redEndPos + greenPos

greenStnName = allStnName[redEndPos:greenEndPos]
greenStnName = list(filter(lambda x: x not in removeWords, greenStnName))

ewl_df = pd.DataFrame(greenStnName, columns=['Name'])
ewl_df['Line'] = 'EWL'
ewl_df.head()
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
      <th>Name</th>
      <th>Line</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Tampines</td>
      <td>EWL</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Simei</td>
      <td>EWL</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Tanah Merah</td>
      <td>EWL</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Bedok</td>
      <td>EWL</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Kembangan</td>
      <td>EWL</td>
    </tr>
  </tbody>
</table>
</div>




```python
## Scraping Changi Airport Line stations from wiki page using xPath
startingXpath = '//table[@class="wikitable"]//tr[66]//text()'
startingXpath = int(re.search(r'[0-9]+', startingXpath)[0])

endingXpath = '//table[@class="wikitable"]//tr[69]//text()' 
endingXpath = int(re.search(r'[0-9]+', endingXpath)[0])

airportPos = endingXpath - startingXpath - 1

airportStnName = ['Tanah Merah']
airportStnName += allStnName[greenEndPos:greenEndPos+airportPos]
airportStnName = list(filter(lambda x: x not in removeWords, airportStnName))

cgl_df = pd.DataFrame(airportStnName, columns=['Name'])
cgl_df['Line'] = 'CGL'
cgl_df.head()
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
      <th>Name</th>
      <th>Line</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Tanah Merah</td>
      <td>CGL</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Changi Airport</td>
      <td>CGL</td>
    </tr>
    <tr>
      <th>2</th>
      <td>HarbourFront</td>
      <td>CGL</td>
    </tr>
  </tbody>
</table>
</div>




```python
## Adding Changi Airport Line to East West Line dataframe
ewl_df1 = ewl_df.iloc[4:]

cgl_df1 = cgl_df.reindex(index=cgl_df.index[::-1])

ewl_df2 = pd.concat([cgl_df1, ewl_df1])
ewl_df2.reset_index(drop=True,inplace=True)

airportStnName = ewl_df2['Name'].tolist()
```


```python
## Scraping Circle Line stations from wiki page using xPath
startingXpath = '//table[@class="wikitable"]//tr[89]//text()'
startingXpath = int(re.search(r'[0-9]+', startingXpath)[0])

endingXpath = '//table[@class="wikitable"]//tr[119]//text()' 
endingXpath = int(re.search(r'[0-9]+', endingXpath)[0])

circlePos = endingXpath - startingXpath - 1

cclPos = allStnName.index("Punggol Coast") + 1

circleStnName = allStnName[cclPos:cclPos+circlePos]
circleStnName = list(filter(lambda x: x not in removeWords, circleStnName))
circleStnName = [c.replace('one-north', 'One-North') for c in circleStnName]

ccl_df = pd.DataFrame(circleStnName, columns=['Name'])
ccl_df['Line'] = 'CCL'
ccl_df.head()
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
      <th>Name</th>
      <th>Line</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Dhoby Ghaut</td>
      <td>CCL</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Bras Basah</td>
      <td>CCL</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Esplanade</td>
      <td>CCL</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Promenade</td>
      <td>CCL</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Nicoll Highway</td>
      <td>CCL</td>
    </tr>
  </tbody>
</table>
</div>




```python
## Scraping Bukit Panjang LRT stations from wiki page using xPath 
bpLine=requests.get('https://en.wikipedia.org/wiki/Bukit_Panjang_LRT_line')
bpTree = html.fromstring(bpLine.content)
bpStnName = bpTree.xpath('//table[@class="wikitable"]//td[2]//text()')
bpStnName = list(filter(lambda x: x not in removeWords, bpStnName))
bpStnName = bpStnName[5:]

bplrt_df = pd.DataFrame(bpStnName, columns=['Name'])
bplrt_df['Line'] = 'BPLRT'
bplrt_df.head()
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
      <th>Name</th>
      <th>Line</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Choa Chu Kang</td>
      <td>BPLRT</td>
    </tr>
    <tr>
      <th>1</th>
      <td>South View</td>
      <td>BPLRT</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Keat Hong</td>
      <td>BPLRT</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Teck Whye</td>
      <td>BPLRT</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Phoenix</td>
      <td>BPLRT</td>
    </tr>
  </tbody>
</table>
</div>




```python
#############################################################
# Converting LTA's MRT & LRT shapefile to Pandas dataframe #
###########################################################
```


```python
!pip install pyshp
```

    Requirement already satisfied: pyshp in /anaconda3/lib/python3.6/site-packages



```python
# import pyshp
import shapefile 

# reading Singapore's MRT/LRT stations from LTA's shapefile
myshp = open('data/MRTLRTStnPtt.shp', "rb")
mydbf = open('data/MRTLRTStnPtt.dbf', "rb")
sf = shapefile.Reader(shp=myshp, dbf=mydbf)
records = sf.shapeRecords()

# checking how many records in the shapefile
print('There are', len(records), 'shape objects in this file')

# view the data in the shapefile
for record in records[:5]:
    print (record.record[0], record.shape.points[0])

```

    There are 183 shape objects in this file
    1 [35782.955299999565, 33560.077600000426]
    2 [16790.746600000188, 36056.30189999938]
    3 [27962.310800000094, 44352.56799999997]
    4 [20081.697399999946, 45214.54790000059]
    5 [26163.47800000012, 30218.819599999115]



```python
# converting shape file from WSG84 to SVY21 (Singapore's Lat Lon standard)
from utils.svy21 import SVY21
svy = SVY21()

## saving all MRT & LRT stations name, & Lat Lon from shapefile to dataframe
latlon = []

for record in records:
    rec = []
    rec.append(record.record[1])
    svylatlon = svy.computeLatLon(record.shape.points[0][1], record.shape.points[0][0])
    rec.append(svylatlon[0])
    rec.append(svylatlon[1])
    latlon.append(rec)

stnlatlon_df = pd.DataFrame(latlon, columns=['StnName', 'Lat', 'Lon'])
stnlatlon_df.head()
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
      <th>StnName</th>
      <th>Lat</th>
      <th>Lon</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>EUNOS MRT STATION</td>
      <td>1.319778</td>
      <td>103.903252</td>
    </tr>
    <tr>
      <th>1</th>
      <td>CHINESE GARDEN MRT STATION</td>
      <td>1.342352</td>
      <td>103.732596</td>
    </tr>
    <tr>
      <th>2</th>
      <td>KHATIB MRT STATION</td>
      <td>1.417383</td>
      <td>103.832980</td>
    </tr>
    <tr>
      <th>3</th>
      <td>KRANJI MRT STATION</td>
      <td>1.425177</td>
      <td>103.762165</td>
    </tr>
    <tr>
      <th>4</th>
      <td>REDHILL MRT STATION</td>
      <td>1.289562</td>
      <td>103.816816</td>
    </tr>
  </tbody>
</table>
</div>




```python
def convertCase(stn):
    newName = stn.title()
    return newName

stnlatlon_df['StnName'] = stnlatlon_df.apply(lambda row: convertCase(row['StnName']), axis=1)
stnlatlon_df.head()
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
      <th>StnName</th>
      <th>Lat</th>
      <th>Lon</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Eunos Mrt Station</td>
      <td>1.319778</td>
      <td>103.903252</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Chinese Garden Mrt Station</td>
      <td>1.342352</td>
      <td>103.732596</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Khatib Mrt Station</td>
      <td>1.417383</td>
      <td>103.832980</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Kranji Mrt Station</td>
      <td>1.425177</td>
      <td>103.762165</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Redhill Mrt Station</td>
      <td>1.289562</td>
      <td>103.816816</td>
    </tr>
  </tbody>
</table>
</div>




```python
def convertLRT(stn):
    newName = stn.replace('Lrt', 'Mrt')
    return newName

stnlatlon_df['StnName'] = stnlatlon_df.apply(lambda row: convertLRT(row['StnName']), axis=1)
stnlatlon_df.head()
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
      <th>StnName</th>
      <th>Lat</th>
      <th>Lon</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Eunos Mrt Station</td>
      <td>1.319778</td>
      <td>103.903252</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Chinese Garden Mrt Station</td>
      <td>1.342352</td>
      <td>103.732596</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Khatib Mrt Station</td>
      <td>1.417383</td>
      <td>103.832980</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Kranji Mrt Station</td>
      <td>1.425177</td>
      <td>103.762165</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Redhill Mrt Station</td>
      <td>1.289562</td>
      <td>103.816816</td>
    </tr>
  </tbody>
</table>
</div>




```python
stnlatlon_df = stnlatlon_df.drop_duplicates(['StnName']) 
stnlatlon_df.tail()
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
      <th>StnName</th>
      <th>Lat</th>
      <th>Lon</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>174</th>
      <td>Buangkok Mrt Station</td>
      <td>1.382877</td>
      <td>103.893121</td>
    </tr>
    <tr>
      <th>177</th>
      <td>Raffles Place Mrt Station</td>
      <td>1.284125</td>
      <td>103.851461</td>
    </tr>
    <tr>
      <th>178</th>
      <td>Holland Village Mrt Station</td>
      <td>1.311834</td>
      <td>103.796191</td>
    </tr>
    <tr>
      <th>180</th>
      <td>Telok Blangah Mrt Station</td>
      <td>1.270753</td>
      <td>103.809748</td>
    </tr>
    <tr>
      <th>181</th>
      <td>Telok Ayer Mrt Station</td>
      <td>1.282289</td>
      <td>103.848302</td>
    </tr>
  </tbody>
</table>
</div>




```python
##############################################################################################
# Mapping and merging all different dataframe together                                      #
# (scrape tweets dataframe, scraped wiki stations dataframe & shapefile stations dataframe) #
############################################################################################
```


```python
mrtTweets = pd.read_csv('smrt_tweet_status_extract.csv')
mrtTweets = mrtTweets.drop(mrtTweets.columns[0], axis=1)

mrtTweets.fillna(value='None',inplace=True)
## Manual data standardization due to very irregular pattern/outlier
mrtTweets['New_Fr_Stn'].replace('Marinabay', 'Marina Bay', inplace = True)
mrtTweets['New_To_Stn'].replace('Marinabay', 'Marina Bay', inplace = True)
mrtTweets['New_Fr_Stn'].replace(['JUR'], 'Jurong East', inplace = True)
mrtTweets['New_To_Stn'].replace(['JUR'], 'Jurong East', inplace = True)
mrtTweets['New_Fr_Stn'].replace(['MSP'], 'Marina South Pier', inplace = True)
mrtTweets['New_To_Stn'].replace(['MSP'], 'Marina South Pier', inplace = True)
mrtTweets['New_Fr_Stn'].replace(['TLK'], 'Tuas Link', inplace = True)
mrtTweets['New_To_Stn'].replace(['TLK'], 'Tuas Link', inplace = True)
mrtTweets['New_Fr_Stn'].replace(['JKN'], 'Joo Koon', inplace = True)
mrtTweets['New_To_Stn'].replace(['JKN'], 'Joo Koon', inplace = True)
mrtTweets['New_Fr_Stn'].replace(['Woodlandwill'], 'Woodlands', inplace = True)
mrtTweets['New_To_Stn'].replace(['Woodlandwill'], 'Woodlands', inplace = True)

mrtTweets.head()
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
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
      <th>Duration</th>
      <th>New_Fr_Stn</th>
      <th>New_To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[BPLRT] Fault cleared. Normal train services a...</td>
      <td>12:54 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>42.0</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[BPLRT]: No train service between #ChoaChuKang...</td>
      <td>12:12 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:12:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[BPLRT] CLEARED: Free regular and bridging bus...</td>
      <td>3:09 AM - 12 Jan 2018</td>
      <td>2018-01-12 03:09:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[BPLRT] UPDATE: Train services on the entire B...</td>
      <td>2:30 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:30:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[BPLRT] UPDATE: Service B on the BPLRT inner l...</td>
      <td>2:04 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:04:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
    </tr>
  </tbody>
</table>
</div>




```python
def parseStn(fromStn):
    newStn = re.sub(r"(\w)([A-Z])", r"\1 \2", str(fromStn))
    newStn += ' dummy'
    return newStn

mrtTweets['new_From_Stn'] = mrtTweets.apply(lambda row: parseStn(str(row['New_Fr_Stn'])), axis=1)
mrtTweets['new_To_Stn'] = mrtTweets.apply(lambda row: parseStn(row['New_To_Stn']), axis=1)

mrtTweets.head()
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
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
      <th>Duration</th>
      <th>New_Fr_Stn</th>
      <th>New_To_Stn</th>
      <th>new_From_Stn</th>
      <th>new_To_Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[BPLRT] Fault cleared. Normal train services a...</td>
      <td>12:54 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>42.0</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
      <td>Choa Chu Kang dummy</td>
      <td>Phoenix dummy</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[BPLRT]: No train service between #ChoaChuKang...</td>
      <td>12:12 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:12:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
      <td>None dummy</td>
      <td>None dummy</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[BPLRT] CLEARED: Free regular and bridging bus...</td>
      <td>3:09 AM - 12 Jan 2018</td>
      <td>2018-01-12 03:09:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
      <td>None dummy</td>
      <td>None dummy</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[BPLRT] UPDATE: Train services on the entire B...</td>
      <td>2:30 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:30:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
      <td>None dummy</td>
      <td>None dummy</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[BPLRT] UPDATE: Service B on the BPLRT inner l...</td>
      <td>2:04 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:04:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
      <td>None dummy</td>
      <td>None dummy</td>
    </tr>
  </tbody>
</table>
</div>




```python
def parseStn(line, fromStn, toStn):
    startStnIndex = ''
    endStnIndex = ''
    affectedStn = []
    if fromStn != 'None dummy':
        fromPat = (re.search(r'[^\s]+', str(fromStn))[0]).title()
        toPat = (re.search(r'[^\s]+', str(toStn))[0]).title()
        if line == 'NSL':
            startStnIndex = nsl_df[nsl_df['Name'].str.match(fromPat)].index[0]
            endStnIndex = nsl_df[nsl_df['Name'].str.match(toPat)].index[0]
            if startStnIndex > endStnIndex:
                affectedStn = redStnName[endStnIndex:startStnIndex+1]
            else:
                affectedStn = redStnName[startStnIndex:endStnIndex+1]
        elif line == 'CCL':
            startStnIndex = ccl_df[ccl_df['Name'].str.match(fromPat)].index[0]
            endStnIndex = ccl_df[ccl_df['Name'].str.match(toPat)].index[0]
            if startStnIndex > endStnIndex:
                affectedStn = circleStnName[endStnIndex:startStnIndex+1]
            else:
                affectedStn = circleStnName[startStnIndex:endStnIndex+1]
        elif (line == 'EWL') & (ewl_df['Name'].str.match(fromPat).any() == True) & (ewl_df['Name'].str.match(toPat).any() == True):
            startStnIndex = ewl_df[(ewl_df['Name'].str.match(fromPat))].index[0]
            endStnIndex = ewl_df[(ewl_df['Name'].str.match(toPat))].index[0]
            if startStnIndex > endStnIndex:
                affectedStn = greenStnName[endStnIndex:startStnIndex+1]
            else:
                affectedStn = greenStnName[startStnIndex:endStnIndex+1]
        elif (line == 'EWL') & (ewl_df2['Name'].str.match(fromPat).any() == True) & (ewl_df2['Name'].str.match(toPat).any() == True):
            startStnIndex = ewl_df2[ewl_df2['Name'].str.match(fromPat)].index[0]
            endStnIndex = ewl_df2[ewl_df2['Name'].str.match(toPat)].index[0]
            if startStnIndex > endStnIndex:
                affectedStn = airportStnName[endStnIndex:startStnIndex+1]
            else:
                affectedStn = airportStnName[startStnIndex:endStnIndex+1]
        elif line == 'BPLRT':
            startStnIndex = bplrt_df[bplrt_df['Name'].str.match(fromPat)].index[0]
            endStnIndex = bplrt_df[bplrt_df['Name'].str.match(toPat)].index[0]
            if startStnIndex > endStnIndex:
                affectedStn = bpStnName[endStnIndex:startStnIndex+1]
            else:
                affectedStn = bpStnName[startStnIndex:endStnIndex+1]
    
    str1 = ','.join(affectedStn)
    return str1

mrtTweets['affectedStn'] = mrtTweets.apply(lambda row: parseStn(row['MRT_Line'], row['new_From_Stn'], row['new_To_Stn']), axis=1)
```


```python
mrtTweets.head()
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
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
      <th>Duration</th>
      <th>New_Fr_Stn</th>
      <th>New_To_Stn</th>
      <th>new_From_Stn</th>
      <th>new_To_Stn</th>
      <th>affectedStn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[BPLRT] Fault cleared. Normal train services a...</td>
      <td>12:54 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>42.0</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
      <td>Choa Chu Kang dummy</td>
      <td>Phoenix dummy</td>
      <td>Choa Chu Kang,South View,Keat Hong,Teck Whye,P...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[BPLRT]: No train service between #ChoaChuKang...</td>
      <td>12:12 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:12:00</td>
      <td>BPLRT</td>
      <td>update</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
      <td>None dummy</td>
      <td>None dummy</td>
      <td></td>
    </tr>
    <tr>
      <th>2</th>
      <td>[BPLRT] CLEARED: Free regular and bridging bus...</td>
      <td>3:09 AM - 12 Jan 2018</td>
      <td>2018-01-12 03:09:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
      <td>None dummy</td>
      <td>None dummy</td>
      <td></td>
    </tr>
    <tr>
      <th>3</th>
      <td>[BPLRT] UPDATE: Train services on the entire B...</td>
      <td>2:30 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:30:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
      <td>None dummy</td>
      <td>None dummy</td>
      <td></td>
    </tr>
    <tr>
      <th>4</th>
      <td>[BPLRT] UPDATE: Service B on the BPLRT inner l...</td>
      <td>2:04 AM - 12 Jan 2018</td>
      <td>2018-01-12 02:04:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>0.0</td>
      <td>None</td>
      <td>None</td>
      <td>None dummy</td>
      <td>None dummy</td>
      <td></td>
    </tr>
  </tbody>
</table>
</div>




```python
breakdown_df = mrtTweets[mrtTweets.affectedStn != '']
breakdown_df.head()
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
      <th>Tweet</th>
      <th>Extracted Date/Time</th>
      <th>Date/Time</th>
      <th>MRT_Line</th>
      <th>Status</th>
      <th>From_Stn</th>
      <th>To_Stn</th>
      <th>Duration</th>
      <th>New_Fr_Stn</th>
      <th>New_To_Stn</th>
      <th>new_From_Stn</th>
      <th>new_To_Stn</th>
      <th>affectedStn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[BPLRT] Fault cleared. Normal train services a...</td>
      <td>12:54 AM - 18 Jan 2018</td>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>42.0</td>
      <td>ChoaChuKang</td>
      <td>Phoenix</td>
      <td>Choa Chu Kang dummy</td>
      <td>Phoenix dummy</td>
      <td>Choa Chu Kang,South View,Keat Hong,Teck Whye,P...</td>
    </tr>
    <tr>
      <th>8</th>
      <td>[BPLRT]\nCLEARED: Train services on the Servic...</td>
      <td>2:15 AM - 27 Jul 2017</td>
      <td>2017-07-27 02:15:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>10.0</td>
      <td>Senja</td>
      <td>Jelapang</td>
      <td>Senja dummy</td>
      <td>Jelapang dummy</td>
      <td>Jelapang,Senja</td>
    </tr>
    <tr>
      <th>11</th>
      <td>[BPLRT] CLEARED: Train service between #ChoaCh...</td>
      <td>10:58 PM - 19 Jul 2017</td>
      <td>2017-07-19 22:58:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>13.0</td>
      <td>ChoaChuKang</td>
      <td>BukitPanjang</td>
      <td>Choa Chu Kang dummy</td>
      <td>Bukit Panjang dummy</td>
      <td>Choa Chu Kang,South View,Keat Hong,Teck Whye,P...</td>
    </tr>
    <tr>
      <th>19</th>
      <td>[BPLRT] UPDATE: Train services have resumed. F...</td>
      <td>4:12 AM - 26 Oct 2016</td>
      <td>2016-10-26 04:12:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>81.0</td>
      <td>BukitPanjang</td>
      <td>Phoenix</td>
      <td>Bukit Panjang dummy</td>
      <td>Phoenix dummy</td>
      <td>Phoenix,Bukit Panjang</td>
    </tr>
    <tr>
      <th>23</th>
      <td>[BPLRT] Svcs on the inner loop (anticlockwise ...</td>
      <td>10:53 PM - 27 Sep 2016</td>
      <td>2016-09-27 22:53:00</td>
      <td>BPLRT</td>
      <td>cleared</td>
      <td>None</td>
      <td>None</td>
      <td>177.0</td>
      <td>ChoaChuKang</td>
      <td>BukitPanjang</td>
      <td>Choa Chu Kang dummy</td>
      <td>Bukit Panjang dummy</td>
      <td>Choa Chu Kang,South View,Keat Hong,Teck Whye,P...</td>
    </tr>
  </tbody>
</table>
</div>




```python
stations = []
def breakdownDF(timestamp, line, affected, duration):
    stn = affected.split(',')
    for i in range(len(stn)):
        breakdownStn = []
        breakdownStn.append(timestamp)
        breakdownStn.append(line)
        breakdownStn.append(stn[i])
        breakdownStn.append(duration)
        stations.append(breakdownStn)
    return breakdownStn
    
affectedStn = breakdown_df.apply(lambda row: breakdownDF(row['Date/Time'], row['MRT_Line'], row['affectedStn'], row['Duration']), axis=1)
```


```python
breakStn_df = pd.DataFrame(stations, columns=['Timestamp', 'Line', 'Stn', 'Duration'])
breakStn_df.head()
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
      <th>Timestamp</th>
      <th>Line</th>
      <th>Stn</th>
      <th>Duration</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>Choa Chu Kang</td>
      <td>42.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>South View</td>
      <td>42.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>Keat Hong</td>
      <td>42.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>Teck Whye</td>
      <td>42.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2018-01-18 00:54:00</td>
      <td>BPLRT</td>
      <td>Phoenix</td>
      <td>42.0</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Which is the most frequent breakdown MRT station
### ================================================
stn_df = breakStn_df.groupby('Stn').count().reset_index().sort_values(['Timestamp'], ascending=False) 
#stn_df = breakStn_df.groupby('Stn').count().reset_index()
stn_df = stn_df.drop(['Line', 'Duration'], axis=1)
stn_df.columns = ['stnName', 'count']
stn_df.head()
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
      <th>stnName</th>
      <th>count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>35</th>
      <td>Jurong East</td>
      <td>28</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Ang Mo Kio</td>
      <td>27</td>
    </tr>
    <tr>
      <th>20</th>
      <td>City Hall</td>
      <td>26</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Bishan</td>
      <td>25</td>
    </tr>
    <tr>
      <th>63</th>
      <td>Raffles Place</td>
      <td>25</td>
    </tr>
  </tbody>
</table>
</div>




```python
def fillName(stn):
    newName = stn + ' Mrt Station'
    return newName

stn_df['stnName'] = stn_df.apply(lambda row: fillName(row['stnName']), axis=1)
stn_df.head()
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
      <th>stnName</th>
      <th>count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>35</th>
      <td>Jurong East Mrt Station</td>
      <td>28</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Ang Mo Kio Mrt Station</td>
      <td>27</td>
    </tr>
    <tr>
      <th>20</th>
      <td>City Hall Mrt Station</td>
      <td>26</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Bishan Mrt Station</td>
      <td>25</td>
    </tr>
    <tr>
      <th>63</th>
      <td>Raffles Place Mrt Station</td>
      <td>25</td>
    </tr>
  </tbody>
</table>
</div>




```python
stn_df.tail()
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
      <th>stnName</th>
      <th>count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>45</th>
      <td>Lorong Chuan Mrt Station</td>
      <td>1</td>
    </tr>
    <tr>
      <th>67</th>
      <td>Serangoon Mrt Station</td>
      <td>1</td>
    </tr>
    <tr>
      <th>33</th>
      <td>Jelapang Mrt Station</td>
      <td>1</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Bartley Mrt Station</td>
      <td>1</td>
    </tr>
    <tr>
      <th>52</th>
      <td>Nicoll Highway Mrt Station</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>




```python
stn_df['stnName'].replace('HarbourFront Mrt Station', 'Harbourfront Mrt Station', inplace = True)
stn_df['stnName'].replace('MacPherson Mrt Station', 'Macpherson Mrt Station', inplace = True)

stations = pd.merge(left=stn_df, right=stnlatlon_df, left_on='stnName', right_on='StnName', how='left')
stations = stations.drop(['StnName'], axis=1)
stations.to_csv('stations.csv')
stations.head()
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
      <th>stnName</th>
      <th>count</th>
      <th>Lat</th>
      <th>Lon</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Jurong East Mrt Station</td>
      <td>28</td>
      <td>1.333152</td>
      <td>103.742286</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Ang Mo Kio Mrt Station</td>
      <td>27</td>
      <td>1.369347</td>
      <td>103.849933</td>
    </tr>
    <tr>
      <th>2</th>
      <td>City Hall Mrt Station</td>
      <td>26</td>
      <td>1.292936</td>
      <td>103.852586</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Bishan Mrt Station</td>
      <td>25</td>
      <td>1.351308</td>
      <td>103.849154</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Raffles Place Mrt Station</td>
      <td>25</td>
      <td>1.284125</td>
      <td>103.851461</td>
    </tr>
  </tbody>
</table>
</div>




```python
#############################################################
# Discovering of insights from scraped dataset/tweets      #
###########################################################
```


```python
### Which MRT line has the most number of station breakdowns
### ========================================================
#%matplotlib inline
import seaborn
breakStn_df.groupby('Line').Stn.count().plot.bar() 
```




    <matplotlib.axes._subplots.AxesSubplot at 0x10e373dd8>




![png](images/output_27_1.png)



```python
breakStn_df.groupby('Line').agg({'Stn':'count'}).reset_index().sort_values(['Stn'], ascending=False) 
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
      <th>Line</th>
      <th>Stn</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>3</th>
      <td>NSL</td>
      <td>404</td>
    </tr>
    <tr>
      <th>2</th>
      <td>EWL</td>
      <td>263</td>
    </tr>
    <tr>
      <th>1</th>
      <td>CCL</td>
      <td>78</td>
    </tr>
    <tr>
      <th>0</th>
      <td>BPLRT</td>
      <td>27</td>
    </tr>
  </tbody>
</table>
</div>




```python
import folium
from folium import plugins

# Ensure Lat Lon are handed as floats
stations['Lat'] = stations['Lat'].astype(float)
stations['Lon'] = stations['Lon'].astype(float)
stations['count'] = stations['count'].astype(float)

# Setting the base folium map to Singapore
sg_map = folium.Map(location=[1.38, 103.8], zoom_start=12)

# List comprehension to make out list of lists
heat_data = [[row['Lat'],row['Lon']] for index, row in stations.iterrows()]

# Plot it on the map
plugins.HeatMap(heat_data).add_to(sg_map)




```python
# mark each station as a point
for index, row in stnlatlon_df.iterrows():
    folium.CircleMarker([row['Lat'], row['Lon']],
                        popup=row['StnName'],
                        radius=3,
                        line_color='#000000',
                        fill_color='#cccccc'
                       ).add_to(sg_map)
    
sg_map
```

![png](images/output_28.png)

<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;charset=utf-8;base64,PCFET0NUWVBFIGh0bWw+CjxoZWFkPiAgICAKICAgIDxtZXRhIGh0dHAtZXF1aXY9ImNvbnRlbnQtdHlwZSIgY29udGVudD0idGV4dC9odG1sOyBjaGFyc2V0PVVURi04IiAvPgogICAgPHNjcmlwdD5MX1BSRUZFUl9DQU5WQVMgPSBmYWxzZTsgTF9OT19UT1VDSCA9IGZhbHNlOyBMX0RJU0FCTEVfM0QgPSBmYWxzZTs8L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2FqYXguZ29vZ2xlYXBpcy5jb20vYWpheC9saWJzL2pxdWVyeS8xLjExLjEvanF1ZXJ5Lm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvanMvYm9vdHN0cmFwLm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvY3NzL2Jvb3RzdHJhcC5taW4uY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLXRoZW1lLm1pbi5jc3MiIC8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vZm9udC1hd2Vzb21lLzQuNi4zL2Nzcy9mb250LWF3ZXNvbWUubWluLmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL3Jhd2dpdC5jb20vcHl0aG9uLXZpc3VhbGl6YXRpb24vZm9saXVtL21hc3Rlci9mb2xpdW0vdGVtcGxhdGVzL2xlYWZsZXQuYXdlc29tZS5yb3RhdGUuY3NzIiAvPgogICAgPHN0eWxlPmh0bWwsIGJvZHkge3dpZHRoOiAxMDAlO2hlaWdodDogMTAwJTttYXJnaW46IDA7cGFkZGluZzogMDt9PC9zdHlsZT4KICAgIDxzdHlsZT4jbWFwIHtwb3NpdGlvbjphYnNvbHV0ZTt0b3A6MDtib3R0b206MDtyaWdodDowO2xlZnQ6MDt9PC9zdHlsZT4KICAgIAogICAgICAgICAgICA8c3R5bGU+ICNtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMgewogICAgICAgICAgICAgICAgcG9zaXRpb24gOiByZWxhdGl2ZTsKICAgICAgICAgICAgICAgIHdpZHRoIDogMTAwLjAlOwogICAgICAgICAgICAgICAgaGVpZ2h0OiAxMDAuMCU7CiAgICAgICAgICAgICAgICBsZWZ0OiAwLjAlOwogICAgICAgICAgICAgICAgdG9wOiAwLjAlOwogICAgICAgICAgICAgICAgfQogICAgICAgICAgICA8L3N0eWxlPgogICAgICAgIAogICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vbGVhZmxldC5naXRodWIuaW8vTGVhZmxldC5oZWF0L2Rpc3QvbGVhZmxldC1oZWF0LmpzIj48L3NjcmlwdD4KPC9oZWFkPgo8Ym9keT4gICAgCiAgICAKICAgICAgICAgICAgPGRpdiBjbGFzcz0iZm9saXVtLW1hcCIgaWQ9Im1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0YyIgPjwvZGl2PgogICAgICAgIAo8L2JvZHk+CjxzY3JpcHQ+ICAgIAogICAgCgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBib3VuZHMgPSBudWxsOwogICAgICAgICAgICAKCiAgICAgICAgICAgIHZhciBtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMgPSBMLm1hcCgKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICdtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMnLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAge2NlbnRlcjogWzEuMzgsMTAzLjhdLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgem9vbTogMTIsCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBtYXhCb3VuZHM6IGJvdW5kcywKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGxheWVyczogW10sCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB3b3JsZENvcHlKdW1wOiBmYWxzZSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGNyczogTC5DUlMuRVBTRzM4NTcKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciB0aWxlX2xheWVyX2I4YTA5NDQzNmM5OTQwNzc5NjFmZmU1MmViOTZkYjFhID0gTC50aWxlTGF5ZXIoCiAgICAgICAgICAgICAgICAnaHR0cHM6Ly97c30udGlsZS5vcGVuc3RyZWV0bWFwLm9yZy97en0ve3h9L3t5fS5wbmcnLAogICAgICAgICAgICAgICAgewogICJhdHRyaWJ1dGlvbiI6IG51bGwsCiAgImRldGVjdFJldGluYSI6IGZhbHNlLAogICJtYXhab29tIjogMTgsCiAgIm1pblpvb20iOiAxLAogICJub1dyYXAiOiBmYWxzZSwKICAic3ViZG9tYWlucyI6ICJhYmMiCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgaGVhdF9tYXBfMTM5Y2Q3Y2YzMzc0NDEwOGFiNGZiNGM3Y2NjNmNmZTQgPSBMLmhlYXRMYXllcigKICAgICAgICAgICAgICAgIFtbMS4zMzMxNTE5NTMyMjc4MzQ5LCAxMDMuNzQyMjg2MjEwNzU4OF0sIFsxLjM2OTM0NjYzNzEyMTcwNywgMTAzLjg0OTkzMjY2MDU1MTg0XSwgWzEuMjkyOTM1NTc2MTc4ODQ0LCAxMDMuODUyNTg1NTU5MDA1NTZdLCBbMS4zNTEzMDgwMTM0NDg5NDQ0LCAxMDMuODQ5MTU0MDg1Njg0MzZdLCBbMS4yODQxMjQ5NDM5MDU5MDY2LCAxMDMuODUxNDYxMzc4NzUyOTNdLCBbMS4zMjc3MTY1MDY0MjcwMjg0LCAxMDMuNjc4Mzc0ODA5ODY1MzZdLCBbMS4zODUzNjEwMjYxNjI4MjE4LCAxMDMuNzQ0MzQwNjEyOTM2NThdLCBbMS40MTczODI3MDM0ODY5Mjc1LCAxMDMuODMyOTc5NTc0NDA1NDFdLCBbMS4zMzI2MjgzMjA0NTYxNDEsIDEwMy44NDc1MDE0MzEwMDMxNF0sIFsxLjM4MTc1NTM3OTQ0NjIwMDUsIDEwMy44NDQ5NDY5NTY0MTc4NF0sIFsxLjMzODYwMzM4NzgzNzg4MDcsIDEwMy43MDYwNjQyODk0Njk4NV0sIFsxLjMzNzU4NjIxNTUzOTU4NTcsIDEwMy42OTczMjExNzk4NTAwMV0sIFsxLjMxMzYwNjQzNTUyNjUwOSwgMTAzLjgzNzgxMDY2MDc4OTkzXSwgWzEuMjk4NzAwNjM5OTYyMjkwNSwgMTAzLjg0NjExNDgyMTMzOTddLCBbMS40Mjk0NDI0MTQxNDQyNDkyLCAxMDMuODM1MDA0NzEyODYxMDldLCBbMS4zMDM5ODAzNDU3Njk2NDczLCAxMDMuODMyMjQxMTE4NzM0NzNdLCBbMS4zMjA0NDAxMjQxODA3NjUzLCAxMDMuODQzODI1Mjg1NDc5NDRdLCBbMS4zMDAyNTkzODgzOTgxOTI2LCAxMDMuODM5MDc0ODI5MDA5M10sIFsxLjM0MDQ3MTAxNzc4NDA3ODIsIDEwMy44NDY3OTc1MTIwNzU0N10sIFsxLjM1ODc2MTE5MDU5NTE4MTcsIDEwMy43NTE4OTMxNjg4MTc5M10sIFsxLjM0OTAzMzQ0MTkxMDM5NywgMTAzLjc0OTU2NjQzMDU5MjA3XSwgWzEuMzA3MTgyNzk5ODgwMzA5NiwgMTAzLjc5MDE5MTIwMTg1ODQ5XSwgWzEuMjc2NDI2Njg4MDgzMjc4LCAxMDMuODU0NTk3NDQ0MjA0MDNdLCBbMS40MzY4NzQ0NjE1NDQ0NzA2LCAxMDMuNzg2NDg0ODAxNTk3OTJdLCBbMS4zOTc1MzQzNTExMjc0MzcsIDEwMy43NDc0MDQ3ODA4ODcwMl0sIFsxLjI5NDg2MDI2NjEyNzc4NzUsIDEwMy44MDU4ODQwMDA1Njc3OV0sIFsxLjQ0OTA1MDE1NDU2MDg3MjcsIDEwMy44MjAwNDU4MDcyMTkzOV0sIFsxLjMxNDk1MzgyNTc1Njg5NDIsIDEwMy43NjUyOTg1NDEwMzY4Nl0sIFsxLjI3OTc2Mzg3NjUxNTM2ODcsIDEwMy44Mzk1NzQ0MDc4ODM5Ml0sIFsxLjQ0MDU4NDMzNDQxOTAzLCAxMDMuODAwOTg3NjY3NDkyXSwgWzEuMzQ0MjU4NTgyNzI4OTY5MywgMTAzLjcyMDk0ODkzNTI5Nl0sIFsxLjQyNTE3NzAzMjEwNDM5MDgsIDEwMy43NjIxNjUwNzc2MjE4NF0sIFsxLjI3NjUyMDU4MDAyNTY4NjIsIDEwMy44NDU4NjM4OTYwMTA4M10sIFsxLjMxODExMTQxNTYyODQ4NjMsIDEwMy44OTMwNjAwMjE4ODU0Ml0sIFsxLjI5OTU1MDA3OTA0MjE5NzksIDEwMy44NTY4Njc5OTk3NTU3Ml0sIFsxLjMxMTQwNDYyNjI1ODQyMTYsIDEwMy43Nzg2Mzc1MDgzMzI1NF0sIFsxLjQzMjUxMzc1Mzk2OTQ2NzIsIDEwMy43NzQwNzExMjc5NjYxNF0sIFsxLjM0MjM1MjE1NDIwODc1MjIsIDEwMy43MzI1OTY0MDQ4NTM2N10sIFsxLjMwMjQzODA2ODY2MDA1NzgsIDEwMy43OTgzMDQxODIzMDg1N10sIFsxLjI4OTU2MjA1OTczNTE3NzMsIDEwMy44MTY4MTYzMzY4MjA2Nl0sIFsxLjMxOTc3ODI4NDg4Njk4ODMsIDEwMy45MDMyNTIxMzMzNTg4Nl0sIFsxLjMyNzE4NjQ2Nzg3MjU1MTYsIDEwMy45NDYzNDYxMjcxNDA3M10sIFsxLjI4NjE5MjcyNjE5MTY2MzIsIDEwMy44MjcwMTkwOTc1OTI0M10sIFsxLjMxMTQ4ODI0Mjk3NTExOTgsIDEwMy44NzEzODYyMDgwMTE3Nl0sIFsxLjMwNzM1NjM1MzQyODQzODcsIDEwMy44NjI4Mjg4NzQ3NjM4XSwgWzEuMzE2NDMxOTQ0OTkwOTIzOSwgMTAzLjg4MjkwNTcxMTMwNDM5XSwgWzEuMzc4NjE4MTc3Nzg2NjQ2NCwgMTAzLjc1ODAzMzc2NTExMjNdLCBbMS4zMTc1MDk5NDUyMjg2NzMsIDEwMy44MDc1ODU3NzI0MjE0OF0sIFsxLjMyMjQyMzMxMjAzMjYyNjUsIDEwMy44MTYxMzEzMTk4MDg4Nl0sIFsxLjI4MjU0MTQ4OTk3NTgzMywgMTAzLjc4MTgxMDExODAxMTIxXSwgWzEuMzIzOTc5MzAyMjIxNTc1MSwgMTAzLjkyOTk4NDE2MTE4NzgyXSwgWzEuMzExODM0MTIzNzM1MDg0LCAxMDMuNzk2MTkxNDkwNTQyNDddLCBbMS4zMjEwMzc1ODIxODQ1OTEsIDEwMy45MTI5NDkwNTQ5NjgwNF0sIFsxLjI5OTc1OTIxMTkyNDk0OSwgMTAzLjc4NzQ1NzE3NDUxNzY4XSwgWzEuMjkzNDYxOTY2NDc3NDY0LCAxMDMuNzg0NjMxMDkzNDgxMzVdLCBbMS4zODAyOTc2MjA1NTEyNDk4LCAxMDMuNzQ1MjkxNDY2Nzk0NTRdLCBbMS4yNzYyMTI4NTYwMTgxNDMyLCAxMDMuNzkxMzQ5OTc5ODM3OF0sIFsxLjM3NjY4NDAxMjc3NzcxMSwgMTAzLjc1MzcxMTg5ODkzMzIzXSwgWzEuMzI2MzQ0NzA0OTk0NDc1NywgMTAzLjg5MDI4NjY5ODAwMjY3XSwgWzEuMzc4NjAyMDk5MzE2OTE4LCAxMDMuNzQ5MDU1MTg2MTQyODhdLCBbMS4zNzc5MDkyMzEzMTU4NzEsIDEwMy43NjMwMTMzMDA1MDI3M10sIFsxLjI3MDc1MjU0NDc5MTIzNjcsIDEwMy44MDk3NDgzMDM1MDk2OF0sIFsxLjMyMTAyNTg4NjU3ODU5NCwgMTAzLjY0OTA3NTE0NTUzNDg1XSwgWzEuMzMzNzI4MjE1NjU0MzY4MiwgMTAzLjgzMDY4OTIxNjgwNDIzXSwgWzEuMjY1NDcxOTczMTk0Njg0OSwgMTAzLjgyMTQ0MjczNTc1ODJdLCBbMS4zMzc2NzM4NDEzNzcyODg2LCAxMDMuODM5NTI5NDg3NDI5MDldLCBbMS4zMTk1MDQ1NDgyNzQxMDgxLCAxMDMuNjYwNTUyNjIzMDQwMzddLCBbMS4yNzIzMzIwNjQ5MjkxNzcsIDEwMy44MDI5MzkyODcwOTY1NV0sIFsxLjM1NzMxMzg3ODY2NjQ4NTYsIDEwMy45ODgzNjQzMjQ4MDkyN10sIFsxLjM0ODcwNjU5NjM1MTIzMzUsIDEwMy44Mzk0MjI3OTkyMzQ5Ml0sIFsxLjM0MzIwMjIyODI2NTE3NTIsIDEwMy45NTMzNzEzNjA2NjQ0NV0sIFsxLjI5Mjg5MDg4NTQyNDk2MzksIDEwMy44NjA4NTI2NTQ5MTQ5XSwgWzEuMzAyODM5OTYyOTY2MTY4OSwgMTAzLjg3NTM1MDAwMTgyODY2XSwgWzEuMzM1NDMyNjU1NTU2MTU2LCAxMDMuODg4MTk0NTI3NzkxNDRdLCBbMS4zMDgzODE5NzI2MDAxMDI1LCAxMDMuODg4NjYyMjcwNTcwNTZdLCBbMS4zODI2OTE2MjkxNTc4NjQ4LCAxMDMuNzYyMzY2ODkyNjY5MTZdLCBbMS4zMDYyMDEyMzgxNjQyNjU0LCAxMDMuODgyNTI3NzQ3NDU0MzJdLCBbMS4yOTMzMjA5NDA5NDQwMDMsIDEwMy44NTU1MDM3NzIxODY5NV0sIFsxLjI5Njg2MTAxOTg2MDM3NywgMTAzLjg1MDY2NzAzODQ0MDE0XSwgWzEuMzUxNjExNTA0MzU4NzQ2OSwgMTAzLjg2NDE1MTYwNDcwMzMxXSwgWzEuMzUwNTk0NTg5Nzc2NTAyNiwgMTAzLjg3MjM2Nzc1NjE1MjRdLCBbMS4zODY3MDIzNTgzNDc2MTU1LCAxMDMuNzY0NTAzMDA0ODc0MzZdLCBbMS4zNDI4Mjc2NzEwNzA3NzI1LCAxMDMuODc5NzU4NTcyODczODVdLCBbMS4yOTk3NjYxNjg2MDA4MjAyLCAxMDMuODYzNjM2Njg1MTUyODhdXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBtaW5PcGFjaXR5OiAwLjUsCiAgICAgICAgICAgICAgICAgICAgbWF4Wm9vbTogMTgsCiAgICAgICAgICAgICAgICAgICAgbWF4OiAxLjAsCiAgICAgICAgICAgICAgICAgICAgcmFkaXVzOiAyNSwKICAgICAgICAgICAgICAgICAgICBibHVyOiAxNSwKICAgICAgICAgICAgICAgICAgICBncmFkaWVudDogbnVsbAogICAgICAgICAgICAgICAgICAgIH0pCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9iZGIxMTA2ZmE4N2M0YTExYjFmZDdlYzZiYzM5ZDk4YiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzE5Nzc4Mjg0ODg2OTg4MywxMDMuOTAzMjUyMTMzMzU4ODZdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2E3NTViYjEwZjg2ZTQzYjM4MmU1NWM2OWYzMGE2Yzg2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzIxNWVlZDBkNjI0NzQwMjE5MWU5NjhiMGI3MGQ4YmI1ID0gJCgnPGRpdiBpZD0iaHRtbF8yMTVlZWQwZDYyNDc0MDIxOTFlOTY4YjBiNzBkOGJiNSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+RXVub3MgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2E3NTViYjEwZjg2ZTQzYjM4MmU1NWM2OWYzMGE2Yzg2LnNldENvbnRlbnQoaHRtbF8yMTVlZWQwZDYyNDc0MDIxOTFlOTY4YjBiNzBkOGJiNSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9iZGIxMTA2ZmE4N2M0YTExYjFmZDdlYzZiYzM5ZDk4Yi5iaW5kUG9wdXAocG9wdXBfYTc1NWJiMTBmODZlNDNiMzgyZTU1YzY5ZjMwYTZjODYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZWNiMmI5OTU0MWM0NDMwY2JmMzY3ZDU1MWU0MWQ4MjkgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM0MjM1MjE1NDIwODc1MjIsMTAzLjczMjU5NjQwNDg1MzY3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yYTIzYjlmMTFlM2I0ODRjYWQ4M2UwNTYwMTNlMWEwNiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jNTkzZDNhYmVhMTg0Y2RlYjlkYzJlYjVmYzJiODA4OSA9ICQoJzxkaXYgaWQ9Imh0bWxfYzU5M2QzYWJlYTE4NGNkZWI5ZGMyZWI1ZmMyYjgwODkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkNoaW5lc2UgR2FyZGVuIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yYTIzYjlmMTFlM2I0ODRjYWQ4M2UwNTYwMTNlMWEwNi5zZXRDb250ZW50KGh0bWxfYzU5M2QzYWJlYTE4NGNkZWI5ZGMyZWI1ZmMyYjgwODkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZWNiMmI5OTU0MWM0NDMwY2JmMzY3ZDU1MWU0MWQ4MjkuYmluZFBvcHVwKHBvcHVwXzJhMjNiOWYxMWUzYjQ4NGNhZDgzZTA1NjAxM2UxYTA2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzdlZmFkMjBlNTVhYzQ2YzhhNmVhNjM2ODZlZGE5M2U4ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS40MTczODI3MDM0ODY5Mjc1LDEwMy44MzI5Nzk1NzQ0MDU0MV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMjc4YTg4NzE2MmUxNDUwNjg0MDE0NjZkYTgyYWRiOTAgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMGRmMjYwNjJkZDk4NGM4ODlhZGIzZWVkZTg1NzY4NzEgPSAkKCc8ZGl2IGlkPSJodG1sXzBkZjI2MDYyZGQ5ODRjODg5YWRiM2VlZGU4NTc2ODcxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5LaGF0aWIgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzI3OGE4ODcxNjJlMTQ1MDY4NDAxNDY2ZGE4MmFkYjkwLnNldENvbnRlbnQoaHRtbF8wZGYyNjA2MmRkOTg0Yzg4OWFkYjNlZWRlODU3Njg3MSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl83ZWZhZDIwZTU1YWM0NmM4YTZlYTYzNjg2ZWRhOTNlOC5iaW5kUG9wdXAocG9wdXBfMjc4YTg4NzE2MmUxNDUwNjg0MDE0NjZkYTgyYWRiOTApOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfNTRhNThmZTAxNzJlNGY3Mzg3YWM4NjZkZDA2M2VlNDUgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjQyNTE3NzAzMjEwNDM5MDgsMTAzLjc2MjE2NTA3NzYyMTg0XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF83ZTI2ZTllYmZlZGE0YWJiODY3OWU4NmZkMTM2ZTMzOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9hOTNmNGMyYjhiNzM0ZDAxOThjOWZmNTJiY2YzMDIwMCA9ICQoJzxkaXYgaWQ9Imh0bWxfYTkzZjRjMmI4YjczNGQwMTk4YzlmZjUyYmNmMzAyMDAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPktyYW5qaSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfN2UyNmU5ZWJmZWRhNGFiYjg2NzllODZmZDEzNmUzMzkuc2V0Q29udGVudChodG1sX2E5M2Y0YzJiOGI3MzRkMDE5OGM5ZmY1MmJjZjMwMjAwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzU0YTU4ZmUwMTcyZTRmNzM4N2FjODY2ZGQwNjNlZTQ1LmJpbmRQb3B1cChwb3B1cF83ZTI2ZTllYmZlZGE0YWJiODY3OWU4NmZkMTM2ZTMzOSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl80Y2U0NTk0NDczZDU0NTEwOTY1YzVlNzU4NzQ3Y2Q2NCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjg5NTYyMDU5NzM1MTc3MywxMDMuODE2ODE2MzM2ODIwNjZdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzdjYWIwOTk4YWM4MzQ5MDQ4YTFkZTFiMWQ3NTZjOTEyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2VmZjI4ZmY1MzhkZTRjZjE5MjY1YTFmYmUxZmE0NzYzID0gJCgnPGRpdiBpZD0iaHRtbF9lZmYyOGZmNTM4ZGU0Y2YxOTI2NWExZmJlMWZhNDc2MyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+UmVkaGlsbCBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfN2NhYjA5OThhYzgzNDkwNDhhMWRlMWIxZDc1NmM5MTIuc2V0Q29udGVudChodG1sX2VmZjI4ZmY1MzhkZTRjZjE5MjY1YTFmYmUxZmE0NzYzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzRjZTQ1OTQ0NzNkNTQ1MTA5NjVjNWU3NTg3NDdjZDY0LmJpbmRQb3B1cChwb3B1cF83Y2FiMDk5OGFjODM0OTA0OGExZGUxYjFkNzU2YzkxMik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9mYzBjZTBmNjdjODQ0YTgwODRkOTcxODhhZDc5MzM0NCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzk3NTM0MzUxMTI3NDM3LDEwMy43NDc0MDQ3ODA4ODcwMl0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNmQwNTNlODlkMzdkNDQxYjkxYmMyYThjMmQ5ZGE0ZDMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNjk0NWUxNjJiNGM0NGYxMjg5YzRkNDI2ZWM2NzQ4YTAgPSAkKCc8ZGl2IGlkPSJodG1sXzY5NDVlMTYyYjRjNDRmMTI4OWM0ZDQyNmVjNjc0OGEwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5ZZXcgVGVlIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF82ZDA1M2U4OWQzN2Q0NDFiOTFiYzJhOGMyZDlkYTRkMy5zZXRDb250ZW50KGh0bWxfNjk0NWUxNjJiNGM0NGYxMjg5YzRkNDI2ZWM2NzQ4YTApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZmMwY2UwZjY3Yzg0NGE4MDg0ZDk3MTg4YWQ3OTMzNDQuYmluZFBvcHVwKHBvcHVwXzZkMDUzZTg5ZDM3ZDQ0MWI5MWJjMmE4YzJkOWRhNGQzKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzY5YjFhNDU0ZWY2MjQ2ZGM4Zjk4MWVhMjdmOTc3ZjVkID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMzc1ODYyMTU1Mzk1ODU3LDEwMy42OTczMjExNzk4NTAwMV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYmJmYTM3MjllZGZhNDhjYmE2ODM2YTQ5ODM2Y2EzMzYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfM2MxMzc2OTRkZWE0NGFhZWI2MTYwOGZhZjAwYWYyN2MgPSAkKCc8ZGl2IGlkPSJodG1sXzNjMTM3Njk0ZGVhNDRhYWViNjE2MDhmYWYwMGFmMjdjIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5QaW9uZWVyIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iYmZhMzcyOWVkZmE0OGNiYTY4MzZhNDk4MzZjYTMzNi5zZXRDb250ZW50KGh0bWxfM2MxMzc2OTRkZWE0NGFhZWI2MTYwOGZhZjAwYWYyN2MpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNjliMWE0NTRlZjYyNDZkYzhmOTgxZWEyN2Y5NzdmNWQuYmluZFBvcHVwKHBvcHVwX2JiZmEzNzI5ZWRmYTQ4Y2JhNjgzNmE0OTgzNmNhMzM2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2I2NzNiOTk4MmYyZjQxOTQ5YTI0MTUxOGZlZWNjZTczID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMDI0MzgwNjg2NjAwNTc4LDEwMy43OTgzMDQxODIzMDg1N10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfM2Y2ZTJmNjFjMDBmNDk2NDhlYzY4NTY3ODI3OGY0YTkgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYzU0MTQxNDE4OGM1NDI2ZGJkZTZkOTBmODZhNzE3NzEgPSAkKCc8ZGl2IGlkPSJodG1sX2M1NDE0MTQxODhjNTQyNmRiZGU2ZDkwZjg2YTcxNzcxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5Db21tb253ZWFsdGggTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzNmNmUyZjYxYzAwZjQ5NjQ4ZWM2ODU2NzgyNzhmNGE5LnNldENvbnRlbnQoaHRtbF9jNTQxNDE0MTg4YzU0MjZkYmRlNmQ5MGY4NmE3MTc3MSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9iNjczYjk5ODJmMmY0MTk0OWEyNDE1MThmZWVjY2U3My5iaW5kUG9wdXAocG9wdXBfM2Y2ZTJmNjFjMDBmNDk2NDhlYzY4NTY3ODI3OGY0YTkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfYjZiM2UxZGY1YjkzNGMwZTlmYjYwM2UxZDFhYmU4Y2YgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMxODExMTQxNTYyODQ4NjMsMTAzLjg5MzA2MDAyMTg4NTQyXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xNTdlMTI2MzU1MmQ0Yjk5OWNhZjgwZGQyMjgxNmU2MCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8yZGZmODk4NTdlODg0MDI0YmUyY2I2OTIxZDJmZTViZiA9ICQoJzxkaXYgaWQ9Imh0bWxfMmRmZjg5ODU3ZTg4NDAyNGJlMmNiNjkyMWQyZmU1YmYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlBheWEgTGViYXIgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzE1N2UxMjYzNTUyZDRiOTk5Y2FmODBkZDIyODE2ZTYwLnNldENvbnRlbnQoaHRtbF8yZGZmODk4NTdlODg0MDI0YmUyY2I2OTIxZDJmZTViZik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9iNmIzZTFkZjViOTM0YzBlOWZiNjAzZTFkMWFiZThjZi5iaW5kUG9wdXAocG9wdXBfMTU3ZTEyNjM1NTJkNGI5OTljYWY4MGRkMjI4MTZlNjApOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfYWQ5ZjFlMTU2N2M5NDAzN2IzNTYyN2JkODVlYmZjOGMgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM0MzIwMjIyODI2NTE3NTIsMTAzLjk1MzM3MTM2MDY2NDQ1XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81NjY5OTg4NDU0NDA0NTcxOTFjOTI4MTUwNDE2MTdlYyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80N2YxMzk5OTE4NmM0NzU5YWViMzRkMjEyZjk4MjUwOSA9ICQoJzxkaXYgaWQ9Imh0bWxfNDdmMTM5OTkxODZjNDc1OWFlYjM0ZDIxMmY5ODI1MDkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNpbWVpIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF81NjY5OTg4NDU0NDA0NTcxOTFjOTI4MTUwNDE2MTdlYy5zZXRDb250ZW50KGh0bWxfNDdmMTM5OTkxODZjNDc1OWFlYjM0ZDIxMmY5ODI1MDkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfYWQ5ZjFlMTU2N2M5NDAzN2IzNTYyN2JkODVlYmZjOGMuYmluZFBvcHVwKHBvcHVwXzU2Njk5ODg0NTQ0MDQ1NzE5MWM5MjgxNTA0MTYxN2VjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzM1ZjU0ZTU1YWMyNjRjYTU4MmZhMzI4N2MwMThkYzYzID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNTMzMDA2ODk2NzY3NDE3LDEwMy45NDUxNDgzNTUxNjU5MV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfY2UzN2NkMDIyZGFiNDA4MGJhNzZkYzc1YzBiNjQ2NDIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYThhNDkzMzdhZTlmNDdjODk3MTBmYTliMzVlNTM4ZjUgPSAkKCc8ZGl2IGlkPSJodG1sX2E4YTQ5MzM3YWU5ZjQ3Yzg5NzEwZmE5YjM1ZTUzOGY1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5UYW1waW5lcyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfY2UzN2NkMDIyZGFiNDA4MGJhNzZkYzc1YzBiNjQ2NDIuc2V0Q29udGVudChodG1sX2E4YTQ5MzM3YWU5ZjQ3Yzg5NzEwZmE5YjM1ZTUzOGY1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzM1ZjU0ZTU1YWMyNjRjYTU4MmZhMzI4N2MwMThkYzYzLmJpbmRQb3B1cChwb3B1cF9jZTM3Y2QwMjJkYWI0MDgwYmE3NmRjNzVjMGI2NDY0Mik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl85YThkYTdhN2QwNjI0MjdlYjRjYWIxNDUxNDFiMTg0YiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuNDQwNTg0MzM0NDE5MDMsMTAzLjgwMDk4NzY2NzQ5Ml0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMWIxMDExYmE2OGU0NDhjN2JiMTRhNTdjYzM3ZmJlNDcgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNWJmZWM5YzA4ZjUxNDFkNWI1ZmMwNzE4ODc5YmE3ZmUgPSAkKCc8ZGl2IGlkPSJodG1sXzViZmVjOWMwOGY1MTQxZDViNWZjMDcxODg3OWJhN2ZlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5BZG1pcmFsdHkgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzFiMTAxMWJhNjhlNDQ4YzdiYjE0YTU3Y2MzN2ZiZTQ3LnNldENvbnRlbnQoaHRtbF81YmZlYzljMDhmNTE0MWQ1YjVmYzA3MTg4NzliYTdmZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl85YThkYTdhN2QwNjI0MjdlYjRjYWIxNDUxNDFiMTg0Yi5iaW5kUG9wdXAocG9wdXBfMWIxMDExYmE2OGU0NDhjN2JiMTRhNTdjYzM3ZmJlNDcpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfYWQ0MzFhZTFlMDA5NGI4ODgwNGUwYWEwYmEyODA5NjYgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjQyOTQ0MjQxNDE0NDI0OTIsMTAzLjgzNTAwNDcxMjg2MTA5XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF82ZDM5MGU5MjUzYzE0NWViOWIzNTc0ODQ3OWFiYzExNiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8zMjlhZDQ5Y2NjNzg0ZjUwYTljYTNlMDBlMzU0NTkwZCA9ICQoJzxkaXYgaWQ9Imh0bWxfMzI5YWQ0OWNjYzc4NGY1MGE5Y2EzZTAwZTM1NDU5MGQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPllpc2h1biBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNmQzOTBlOTI1M2MxNDVlYjliMzU3NDg0NzlhYmMxMTYuc2V0Q29udGVudChodG1sXzMyOWFkNDljY2M3ODRmNTBhOWNhM2UwMGUzNTQ1OTBkKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2FkNDMxYWUxZTAwOTRiODg4MDRlMGFhMGJhMjgwOTY2LmJpbmRQb3B1cChwb3B1cF82ZDM5MGU5MjUzYzE0NWViOWIzNTc0ODQ3OWFiYzExNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl80MjE0ZDIyNmJlMzk0OWU5ODY2YmFjNWVmNDc1YjE4NyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzcyOTgzMTA2OTEzOTk4OCwxMDMuOTQ5MjY3Nzg0NzQzMTFdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2VhM2ZhN2JiMjBiZDQyZWQ4ZTFkYjExZTEwZjg1MzlmID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2VlOWNkYWMwNWRjNDRjNDVhMDU1MTgzN2IyNTAzYmFiID0gJCgnPGRpdiBpZD0iaHRtbF9lZTljZGFjMDVkYzQ0YzQ1YTA1NTE4MzdiMjUwM2JhYiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+UGFzaXIgUmlzIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lYTNmYTdiYjIwYmQ0MmVkOGUxZGIxMWUxMGY4NTM5Zi5zZXRDb250ZW50KGh0bWxfZWU5Y2RhYzA1ZGM0NGM0NWEwNTUxODM3YjI1MDNiYWIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNDIxNGQyMjZiZTM5NDllOTg2NmJhYzVlZjQ3NWIxODcuYmluZFBvcHVwKHBvcHVwX2VhM2ZhN2JiMjBiZDQyZWQ4ZTFkYjExZTEwZjg1MzlmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzRmNjllNWI1MjEwZjQ5ZjE5YjFjNGFjNzc0MTJkMzIwID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMTQ5NTM4MjU3NTY4OTQyLDEwMy43NjUyOTg1NDEwMzY4Nl0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMDE5YjQ4NDYwMWY1NDMwYmJmNzRjYjZhMzBmN2EwMTQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZTEyZDhjZDFiODIyNGQ2Nzg0ZWI4MmEyZmM0Mjc1YzQgPSAkKCc8ZGl2IGlkPSJodG1sX2UxMmQ4Y2QxYjgyMjRkNjc4NGViODJhMmZjNDI3NWM0IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5DbGVtZW50aSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMDE5YjQ4NDYwMWY1NDMwYmJmNzRjYjZhMzBmN2EwMTQuc2V0Q29udGVudChodG1sX2UxMmQ4Y2QxYjgyMjRkNjc4NGViODJhMmZjNDI3NWM0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzRmNjllNWI1MjEwZjQ5ZjE5YjFjNGFjNzc0MTJkMzIwLmJpbmRQb3B1cChwb3B1cF8wMTliNDg0NjAxZjU0MzBiYmY3NGNiNmEzMGY3YTAxNCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9hZGM0OTE2ZmRkMzM0NDA2ODBjYmUwYjYxNTk5Mjk4YSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzE2NDMxOTQ0OTkwOTIzOSwxMDMuODgyOTA1NzExMzA0MzldLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzA1NmNjZmY5YWRlYjRjNWJiYzIzNTlmYmRhYWQ5ZTI1ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2E5Y2E3MjY1ZWQ3YzRhZGU4ZDQ3ZjA5MzVmMTI3M2YzID0gJCgnPGRpdiBpZD0iaHRtbF9hOWNhNzI2NWVkN2M0YWRlOGQ0N2YwOTM1ZjEyNzNmMyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+QWxqdW5pZWQgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzA1NmNjZmY5YWRlYjRjNWJiYzIzNTlmYmRhYWQ5ZTI1LnNldENvbnRlbnQoaHRtbF9hOWNhNzI2NWVkN2M0YWRlOGQ0N2YwOTM1ZjEyNzNmMyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9hZGM0OTE2ZmRkMzM0NDA2ODBjYmUwYjYxNTk5Mjk4YS5iaW5kUG9wdXAocG9wdXBfMDU2Y2NmZjlhZGViNGM1YmJjMjM1OWZiZGFhZDllMjUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMTkxYTg4NGMyYWI2NGEyMmI1NGNjNjQ0NTJiM2M5NDAgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM2OTM0NjYzNzEyMTcwNywxMDMuODQ5OTMyNjYwNTUxODRdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzZlZTBhYTZmOTlmMjQ3ZGQ4NDY5YWYyMjA0ZDFmYTFiID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2EyYzEzM2E4MDVjMDQwMWQ5MDhjY2VjM2E0MWYyMDlkID0gJCgnPGRpdiBpZD0iaHRtbF9hMmMxMzNhODA1YzA0MDFkOTA4Y2NlYzNhNDFmMjA5ZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+QW5nIE1vIEtpbyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNmVlMGFhNmY5OWYyNDdkZDg0NjlhZjIyMDRkMWZhMWIuc2V0Q29udGVudChodG1sX2EyYzEzM2E4MDVjMDQwMWQ5MDhjY2VjM2E0MWYyMDlkKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzE5MWE4ODRjMmFiNjRhMjJiNTRjYzY0NDUyYjNjOTQwLmJpbmRQb3B1cChwb3B1cF82ZWUwYWE2Zjk5ZjI0N2RkODQ2OWFmMjIwNGQxZmExYik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8xMGEyOWU5MDk3OTQ0NjU3Yjc2ZGI5YTliNzMyYTE4NSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzM0NTQ5MTExMTcyNDIxMywxMDMuOTYxNTQ3ODc3MTY1MV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZTYyZjNjMTQ0NDJjNDc2NGFiOGRiYjdhY2E1ODA0NTUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfN2YyMTMyOTMyMGJiNDk4NmIxZWU2ZjhhMDBhN2Y2NGUgPSAkKCc8ZGl2IGlkPSJodG1sXzdmMjEzMjkzMjBiYjQ5ODZiMWVlNmY4YTAwYTdmNjRlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5FeHBvIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lNjJmM2MxNDQ0MmM0NzY0YWI4ZGJiN2FjYTU4MDQ1NS5zZXRDb250ZW50KGh0bWxfN2YyMTMyOTMyMGJiNDk4NmIxZWU2ZjhhMDBhN2Y2NGUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMTBhMjllOTA5Nzk0NDY1N2I3NmRiOWE5YjczMmExODUuYmluZFBvcHVwKHBvcHVwX2U2MmYzYzE0NDQyYzQ3NjRhYjhkYmI3YWNhNTgwNDU1KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzg1Y2VlYTYwNGM1NDRkMmQ5NjUzOGZjMDU2YjlkN2ZlID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zODUzNjEwMjYxNjI4MjE4LDEwMy43NDQzNDA2MTI5MzY1OF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMGQ3ODQ5OGI1YWRhNDgwZWEyZTFlNzU3MmE4NGQ4ZDIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfODMxNTlhZmY1NmY1NGExNjhmMzZlNmVhZDM0NmMwM2UgPSAkKCc8ZGl2IGlkPSJodG1sXzgzMTU5YWZmNTZmNTRhMTY4ZjM2ZTZlYWQzNDZjMDNlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5DaG9hIENodSBLYW5nIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wZDc4NDk4YjVhZGE0ODBlYTJlMWU3NTcyYTg0ZDhkMi5zZXRDb250ZW50KGh0bWxfODMxNTlhZmY1NmY1NGExNjhmMzZlNmVhZDM0NmMwM2UpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfODVjZWVhNjA0YzU0NGQyZDk2NTM4ZmMwNTZiOWQ3ZmUuYmluZFBvcHVwKHBvcHVwXzBkNzg0OThiNWFkYTQ4MGVhMmUxZTc1NzJhODRkOGQyKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2IxYzIzNTYzYmIyNDQ5ZDA4ZGU0MWRiNjc1MzM1ODM3ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMzMxNTE5NTMyMjc4MzQ5LDEwMy43NDIyODYyMTA3NTg4XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yMjlkYzJiNzFkZTQ0YWNjYTBlZTM2OGY4Y2JkOTQ0MCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8wMTM2NDYxZThlMjc0MDY2OTZhZTZiZDk5MWVlZmQ3NyA9ICQoJzxkaXYgaWQ9Imh0bWxfMDEzNjQ2MWU4ZTI3NDA2Njk2YWU2YmQ5OTFlZWZkNzciIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkp1cm9uZyBFYXN0IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yMjlkYzJiNzFkZTQ0YWNjYTBlZTM2OGY4Y2JkOTQ0MC5zZXRDb250ZW50KGh0bWxfMDEzNjQ2MWU4ZTI3NDA2Njk2YWU2YmQ5OTFlZWZkNzcpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfYjFjMjM1NjNiYjI0NDlkMDhkZTQxZGI2NzUzMzU4MzcuYmluZFBvcHVwKHBvcHVwXzIyOWRjMmI3MWRlNDRhY2NhMGVlMzY4ZjhjYmQ5NDQwKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzQwYzZjMzQ5MTQzZjQ5MzQ4MDMwODk4ZTljMzUwZDVmID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yOTQ4NjAyNjYxMjc3ODc1LDEwMy44MDU4ODQwMDA1Njc3OV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNjRiMzc0NWU0NGVlNGVkMjg5OGJjYTM0MjM2MjVkYjEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZGJjNzg1ODA2YTQzNDM4OWIxM2I5MGI3MzFkNDdjOGYgPSAkKCc8ZGl2IGlkPSJodG1sX2RiYzc4NTgwNmE0MzQzODliMTNiOTBiNzMxZDQ3YzhmIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5RdWVlbnN0b3duIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF82NGIzNzQ1ZTQ0ZWU0ZWQyODk4YmNhMzQyMzYyNWRiMS5zZXRDb250ZW50KGh0bWxfZGJjNzg1ODA2YTQzNDM4OWIxM2I5MGI3MzFkNDdjOGYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNDBjNmMzNDkxNDNmNDkzNDgwMzA4OThlOWMzNTBkNWYuYmluZFBvcHVwKHBvcHVwXzY0YjM3NDVlNDRlZTRlZDI4OThiY2EzNDIzNjI1ZGIxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzU2NjNkMzAwNDNhYjQxZGM4YzZmNGUwZjJkZDhjODAwID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNDA0NjMwOTg5ODUxMTMyLDEwMy42MzY4NzgzNDU3MDUyXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jMmY5MWE1OWFjMDk0NGM2YjM4M2RhYjhiN2U1ZDc3ZiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80YTliZTJkNjhjYjA0ZmZkOTkyNDY3NDczNzY0ZWRmYSA9ICQoJzxkaXYgaWQ9Imh0bWxfNGE5YmUyZDY4Y2IwNGZmZDk5MjQ2NzQ3Mzc2NGVkZmEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlR1YXMgTGluayBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYzJmOTFhNTlhYzA5NDRjNmIzODNkYWI4YjdlNWQ3N2Yuc2V0Q29udGVudChodG1sXzRhOWJlMmQ2OGNiMDRmZmQ5OTI0Njc0NzM3NjRlZGZhKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzU2NjNkMzAwNDNhYjQxZGM4YzZmNGUwZjJkZDhjODAwLmJpbmRQb3B1cChwb3B1cF9jMmY5MWE1OWFjMDk0NGM2YjM4M2RhYjhiN2U1ZDc3Zik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8wNWViMTE2NGM1ZTc0ZGExYTBhZGIyODI1ZjdkNzJhMSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzE5NTA0NTQ4Mjc0MTA4MSwxMDMuNjYwNTUyNjIzMDQwMzddLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzI0NjE3NGE2NjhmYjQxN2ZhZGUwOGNhODM0NzdlNjQ0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzkxZTI0NDA2YTkxNjQyMmVhYzliMGFhZmFmZjE5NTUxID0gJCgnPGRpdiBpZD0iaHRtbF85MWUyNDQwNmE5MTY0MjJlYWM5YjBhYWZhZmYxOTU1MSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+R3VsIENpcmNsZSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMjQ2MTc0YTY2OGZiNDE3ZmFkZTA4Y2E4MzQ3N2U2NDQuc2V0Q29udGVudChodG1sXzkxZTI0NDA2YTkxNjQyMmVhYzliMGFhZmFmZjE5NTUxKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzA1ZWIxMTY0YzVlNzRkYTFhMGFkYjI4MjVmN2Q3MmExLmJpbmRQb3B1cChwb3B1cF8yNDYxNzRhNjY4ZmI0MTdmYWRlMDhjYTgzNDc3ZTY0NCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9hM2VjZDMyNGY3MmQ0MWNmODEyNDBkNWU3YWMwYmNiNCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuNDM2ODc0NDYxNTQ0NDcwNiwxMDMuNzg2NDg0ODAxNTk3OTJdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzczYmJjNzFmZTAzOTRiNzZiZjhjYTEyZWFlMDIzMzhhID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzRlYWM3NWU0ZDlmZjRmZDhiMzkwMmU3MGMwNWYyZjMzID0gJCgnPGRpdiBpZD0iaHRtbF80ZWFjNzVlNGQ5ZmY0ZmQ4YjM5MDJlNzBjMDVmMmYzMyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+V29vZGxhbmRzIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF83M2JiYzcxZmUwMzk0Yjc2YmY4Y2ExMmVhZTAyMzM4YS5zZXRDb250ZW50KGh0bWxfNGVhYzc1ZTRkOWZmNGZkOGIzOTAyZTcwYzA1ZjJmMzMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfYTNlY2QzMjRmNzJkNDFjZjgxMjQwZDVlN2FjMGJjYjQuYmluZFBvcHVwKHBvcHVwXzczYmJjNzFmZTAzOTRiNzZiZjhjYTEyZWFlMDIzMzhhKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzRhNzAyZTFkOGZjNzQ2ZGM4MzIyYjU2ZGRlOTAzNTM0ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMjEwMjU4ODY1Nzg1OTQsMTAzLjY0OTA3NTE0NTUzNDg1XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81YmZjZGY5MmNhMGQ0OTNkODQyMmYyNzA1N2M2NjgwMCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82NzA2ZGY2ODkwYWI0ODYwODk5MDhiN2ZkNTg2NjZkNSA9ICQoJzxkaXYgaWQ9Imh0bWxfNjcwNmRmNjg5MGFiNDg2MDg5OTA4YjdmZDU4NjY2ZDUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlR1YXMgQ3Jlc2NlbnQgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzViZmNkZjkyY2EwZDQ5M2Q4NDIyZjI3MDU3YzY2ODAwLnNldENvbnRlbnQoaHRtbF82NzA2ZGY2ODkwYWI0ODYwODk5MDhiN2ZkNTg2NjZkNSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl80YTcwMmUxZDhmYzc0NmRjODMyMmI1NmRkZTkwMzUzNC5iaW5kUG9wdXAocG9wdXBfNWJmY2RmOTJjYTBkNDkzZDg0MjJmMjcwNTdjNjY4MDApOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZWFhNWMyYTBmMmYwNDIyNzhhNmE5YTdiYjlkYzViYWUgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMxMTQ4ODI0Mjk3NTExOTgsMTAzLjg3MTM4NjIwODAxMTc2XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iNGM5OGIzYmMyMjk0M2YyYjNjZWU3MzlkMGUyNWUxZCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85MzQ1YTRkZWU3ZTM0N2EyOWYyOWRjZGYyM2Q4MzAzMSA9ICQoJzxkaXYgaWQ9Imh0bWxfOTM0NWE0ZGVlN2UzNDdhMjlmMjlkY2RmMjNkODMwMzEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkthbGxhbmcgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2I0Yzk4YjNiYzIyOTQzZjJiM2NlZTczOWQwZTI1ZTFkLnNldENvbnRlbnQoaHRtbF85MzQ1YTRkZWU3ZTM0N2EyOWYyOWRjZGYyM2Q4MzAzMSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9lYWE1YzJhMGYyZjA0MjI3OGE2YTlhN2JiOWRjNWJhZS5iaW5kUG9wdXAocG9wdXBfYjRjOThiM2JjMjI5NDNmMmIzY2VlNzM5ZDBlMjVlMWQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfYjA1NGVmZTM4NjlkNGIyYjk2OWRiYWUxYTVhMTRmZGMgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMyOTk4ODQyMjM4NTg3NjcsMTAzLjYzOTYxMzcyMzM0OTg3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iYThlMWY5NGY3NTg0MDMwYmY1ZTkwOWRlYjk0ODY3MyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF84MjYyN2MwMGI2NzI0OGE2OWE3ZmM1NjdmMTQ1NTVlMCA9ICQoJzxkaXYgaWQ9Imh0bWxfODI2MjdjMDBiNjcyNDhhNjlhN2ZjNTY3ZjE0NTU1ZTAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlR1YXMgV2VzdCBSb2FkIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iYThlMWY5NGY3NTg0MDMwYmY1ZTkwOWRlYjk0ODY3My5zZXRDb250ZW50KGh0bWxfODI2MjdjMDBiNjcyNDhhNjlhN2ZjNTY3ZjE0NTU1ZTApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfYjA1NGVmZTM4NjlkNGIyYjk2OWRiYWUxYTVhMTRmZGMuYmluZFBvcHVwKHBvcHVwX2JhOGUxZjk0Zjc1ODQwMzBiZjVlOTA5ZGViOTQ4NjczKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2NjNTc0OWM4M2M1MzRlOGM5MTBmZGM3M2UxOTUyNzM1ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMDcxODI3OTk4ODAzMDk2LDEwMy43OTAxOTEyMDE4NTg0OV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNzFhZTNjMjlhNTgxNGE2OGI0MGY5M2E3Y2E2NGZjYmIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNjAzNDJiN2QyNjNkNDAyYjgzZTA3MGMzNGMxMzEzMDUgPSAkKCc8ZGl2IGlkPSJodG1sXzYwMzQyYjdkMjYzZDQwMmI4M2UwNzBjMzRjMTMxMzA1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5CdW9uYSBWaXN0YSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNzFhZTNjMjlhNTgxNGE2OGI0MGY5M2E3Y2E2NGZjYmIuc2V0Q29udGVudChodG1sXzYwMzQyYjdkMjYzZDQwMmI4M2UwNzBjMzRjMTMxMzA1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2NjNTc0OWM4M2M1MzRlOGM5MTBmZGM3M2UxOTUyNzM1LmJpbmRQb3B1cChwb3B1cF83MWFlM2MyOWE1ODE0YTY4YjQwZjkzYTdjYTY0ZmNiYik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8xZGMwMGM1YzJjMDA0NTY1ODUwZmUzNzA5OWVlYTI3ZCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzExNDA0NjI2MjU4NDIxNiwxMDMuNzc4NjM3NTA4MzMyNTRdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzcyZWE2NWU5ZmNmNjQ4Zjk5YWRiYjYzMjAxMzkzZmVmID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzA0MGY4YThhN2IxNzQ5ZTc4OGZjMjViZTJjYmE4OWI4ID0gJCgnPGRpdiBpZD0iaHRtbF8wNDBmOGE4YTdiMTc0OWU3ODhmYzI1YmUyY2JhODliOCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+RG92ZXIgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzcyZWE2NWU5ZmNmNjQ4Zjk5YWRiYjYzMjAxMzkzZmVmLnNldENvbnRlbnQoaHRtbF8wNDBmOGE4YTdiMTc0OWU3ODhmYzI1YmUyY2JhODliOCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl8xZGMwMGM1YzJjMDA0NTY1ODUwZmUzNzA5OWVlYTI3ZC5iaW5kUG9wdXAocG9wdXBfNzJlYTY1ZTlmY2Y2NDhmOTlhZGJiNjMyMDEzOTNmZWYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfYTQxNGQ2ZGI3MTMxNDI3Yjg3MTAxOThkZTFmZDg4ZTkgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjQ0OTA1MDE1NDU2MDg3MjcsMTAzLjgyMDA0NTgwNzIxOTM5XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85NDJmYzVmMjY3NTA0MDdhOTk4NDc2MGY2ZTNmOTk0OCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8zMzFmMTM0NTIzYjg0YWRmYThkMTQwNmZhYTRmYzlhMCA9ICQoJzxkaXYgaWQ9Imh0bWxfMzMxZjEzNDUyM2I4NGFkZmE4ZDE0MDZmYWE0ZmM5YTAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNlbWJhd2FuZyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOTQyZmM1ZjI2NzUwNDA3YTk5ODQ3NjBmNmUzZjk5NDguc2V0Q29udGVudChodG1sXzMzMWYxMzQ1MjNiODRhZGZhOGQxNDA2ZmFhNGZjOWEwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2E0MTRkNmRiNzEzMTQyN2I4NzEwMTk4ZGUxZmQ4OGU5LmJpbmRQb3B1cChwb3B1cF85NDJmYzVmMjY3NTA0MDdhOTk4NDc2MGY2ZTNmOTk0OCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9iYjg5NGE3MDA2ZWM0NTEwYjI2YzE2OWNiNzI5Yzk3MiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzIxMDM3NTgyMTg0NTkxLDEwMy45MTI5NDkwNTQ5NjgwNF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMGQwNjMwZDI3MGYxNDhkNmE1Zjc4ZjdlNjMxMmJkNjEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZjRiODcxOTI3YzA4NDI1OGE2ZWVkNjdkYWNhYWMwNjYgPSAkKCc8ZGl2IGlkPSJodG1sX2Y0Yjg3MTkyN2MwODQyNThhNmVlZDY3ZGFjYWFjMDY2IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5LZW1iYW5nYW4gTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzBkMDYzMGQyNzBmMTQ4ZDZhNWY3OGY3ZTYzMTJiZDYxLnNldENvbnRlbnQoaHRtbF9mNGI4NzE5MjdjMDg0MjU4YTZlZWQ2N2RhY2FhYzA2Nik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9iYjg5NGE3MDA2ZWM0NTEwYjI2YzE2OWNiNzI5Yzk3Mi5iaW5kUG9wdXAocG9wdXBfMGQwNjMwZDI3MGYxNDhkNmE1Zjc4ZjdlNjMxMmJkNjEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMDVhMzY0ZjU4NDBlNDkwYmIwYjJiMDY0MGI2M2JmY2UgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM0OTAzMzQ0MTkxMDM5NywxMDMuNzQ5NTY2NDMwNTkyMDddLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzAxYTQ2MTg2MmNhNjQ2MzI5NDQyOGM5Yjk2NjUxMmE4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2NmZDg1N2I0ODg5NzQxM2Q4ZjYxOWZlYmE3YzBjNWNkID0gJCgnPGRpdiBpZD0iaHRtbF9jZmQ4NTdiNDg4OTc0MTNkOGY2MTlmZWJhN2MwYzVjZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+QnVraXQgQmF0b2sgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzAxYTQ2MTg2MmNhNjQ2MzI5NDQyOGM5Yjk2NjUxMmE4LnNldENvbnRlbnQoaHRtbF9jZmQ4NTdiNDg4OTc0MTNkOGY2MTlmZWJhN2MwYzVjZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl8wNWEzNjRmNTg0MGU0OTBiYjBiMmIwNjQwYjYzYmZjZS5iaW5kUG9wdXAocG9wdXBfMDFhNDYxODYyY2E2NDYzMjk0NDI4YzliOTY2NTEyYTgpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMzUxODM4OTlkZTA5NDM0OWJmMzNkYzI3MGVhNmZiYjggPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMzODYwMzM4NzgzNzg4MDcsMTAzLjcwNjA2NDI4OTQ2OTg1XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85YWMzMGU4ZDVhZmE0YTgwYmNlZGQ4NjJhNzVmYmQzNSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9hYjY2ZTMxNzljYzk0MzVjODQ5MDcyZTg5NzdlMjE1OSA9ICQoJzxkaXYgaWQ9Imh0bWxfYWI2NmUzMTc5Y2M5NDM1Yzg0OTA3MmU4OTc3ZTIxNTkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkJvb24gTGF5IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF85YWMzMGU4ZDVhZmE0YTgwYmNlZGQ4NjJhNzVmYmQzNS5zZXRDb250ZW50KGh0bWxfYWI2NmUzMTc5Y2M5NDM1Yzg0OTA3MmU4OTc3ZTIxNTkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMzUxODM4OTlkZTA5NDM0OWJmMzNkYzI3MGVhNmZiYjguYmluZFBvcHVwKHBvcHVwXzlhYzMwZThkNWFmYTRhODBiY2VkZDg2MmE3NWZiZDM1KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2UzY2M0M2IxYjc4NzQzMmE4ZmJlNmNlZTVkYzliM2I0ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNTg3NjExOTA1OTUxODE3LDEwMy43NTE4OTMxNjg4MTc5M10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZDI1ZDAxNzEzYzcyNDM2ZTg1NGRlYzBlMzAzOThiY2UgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYzEwOWU2N2ZiMDc2NDc2NThkMDA0MDJkZTgzYzMzOTkgPSAkKCc8ZGl2IGlkPSJodG1sX2MxMDllNjdmYjA3NjQ3NjU4ZDAwNDAyZGU4M2MzMzk5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5CdWtpdCBHb21iYWsgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2QyNWQwMTcxM2M3MjQzNmU4NTRkZWMwZTMwMzk4YmNlLnNldENvbnRlbnQoaHRtbF9jMTA5ZTY3ZmIwNzY0NzY1OGQwMDQwMmRlODNjMzM5OSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9lM2NjNDNiMWI3ODc0MzJhOGZiZTZjZWU1ZGM5YjNiNC5iaW5kUG9wdXAocG9wdXBfZDI1ZDAxNzEzYzcyNDM2ZTg1NGRlYzBlMzAzOThiY2UpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfYTgyN2UyNTJlNGE2NGQ3ZWFkYzFkODc0NWQxYzFhMDAgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjQzMjUxMzc1Mzk2OTQ2NzIsMTAzLjc3NDA3MTEyNzk2NjE0XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9kZTEwNDA4MmEyY2Q0ZDU2YTJkNDNlMzlmZGQ4ZmQ3MyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8xYThjOTE3Y2I0MWY0NjU3YWJmMjVjOGRhZGVhODhkYyA9ICQoJzxkaXYgaWQ9Imh0bWxfMWE4YzkxN2NiNDFmNDY1N2FiZjI1YzhkYWRlYTg4ZGMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPk1hcnNpbGluZyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZGUxMDQwODJhMmNkNGQ1NmEyZDQzZTM5ZmRkOGZkNzMuc2V0Q29udGVudChodG1sXzFhOGM5MTdjYjQxZjQ2NTdhYmYyNWM4ZGFkZWE4OGRjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2E4MjdlMjUyZTRhNjRkN2VhZGMxZDg3NDVkMWMxYTAwLmJpbmRQb3B1cChwb3B1cF9kZTEwNDA4MmEyY2Q0ZDU2YTJkNDNlMzlmZGQ4ZmQ3Myk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9iYjY0NDVkNDZlMzk0YThmODMyYzk4ZTY5NDRiN2VhMSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzIzOTc5MzAyMjIxNTc1MSwxMDMuOTI5OTg0MTYxMTg3ODJdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2UyODY1NzI3MDA0NTQ1ODg5ZDUyZDY0NjBlN2RiZDdlID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzY4NTVhNTIzZTI4ZjQzMDBhYmJiNzYyOTZiZWQ4MmZkID0gJCgnPGRpdiBpZD0iaHRtbF82ODU1YTUyM2UyOGY0MzAwYWJiYjc2Mjk2YmVkODJmZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+QmVkb2sgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2UyODY1NzI3MDA0NTQ1ODg5ZDUyZDY0NjBlN2RiZDdlLnNldENvbnRlbnQoaHRtbF82ODU1YTUyM2UyOGY0MzAwYWJiYjc2Mjk2YmVkODJmZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9iYjY0NDVkNDZlMzk0YThmODMyYzk4ZTY5NDRiN2VhMS5iaW5kUG9wdXAocG9wdXBfZTI4NjU3MjcwMDQ1NDU4ODlkNTJkNjQ2MGU3ZGJkN2UpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZTQ0MDc3MTA4YzM3NDZhYTgzZDM4MTVjZjBmM2QzMWYgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMyNzcxNjUwNjQyNzAyODQsMTAzLjY3ODM3NDgwOTg2NTM2XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iNDY1NWYwOWUxNTY0NzMzOTY0YjEzMTRkZmNmYTY3ZiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85ZjI3OGM4MWQ2YTY0OThmYTVjZmQ4MmQ2ZWNlYjQ2NSA9ICQoJzxkaXYgaWQ9Imh0bWxfOWYyNzhjODFkNmE2NDk4ZmE1Y2ZkODJkNmVjZWI0NjUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkpvbyBLb29uIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iNDY1NWYwOWUxNTY0NzMzOTY0YjEzMTRkZmNmYTY3Zi5zZXRDb250ZW50KGh0bWxfOWYyNzhjODFkNmE2NDk4ZmE1Y2ZkODJkNmVjZWI0NjUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZTQ0MDc3MTA4YzM3NDZhYTgzZDM4MTVjZjBmM2QzMWYuYmluZFBvcHVwKHBvcHVwX2I0NjU1ZjA5ZTE1NjQ3MzM5NjRiMTMxNGRmY2ZhNjdmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzhlZjNjMjdmNGI4NjQ4NDdiMzE3OWIyYTZjOWQyZWMzID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNDQyNTg1ODI3Mjg5NjkzLDEwMy43MjA5NDg5MzUyOTZdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2Q5ODVjN2YzNGIxYzQ1NGFhNWNkZTNlY2MwZTgxZmIxID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzFiZTA2MjkyMjUzYzRmMDRhNmY5NDQwZTE5ZDk1MjM2ID0gJCgnPGRpdiBpZD0iaHRtbF8xYmUwNjI5MjI1M2M0ZjA0YTZmOTQ0MGUxOWQ5NTIzNiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+TGFrZXNpZGUgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2Q5ODVjN2YzNGIxYzQ1NGFhNWNkZTNlY2MwZTgxZmIxLnNldENvbnRlbnQoaHRtbF8xYmUwNjI5MjI1M2M0ZjA0YTZmOTQ0MGUxOWQ5NTIzNik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl84ZWYzYzI3ZjRiODY0ODQ3YjMxNzliMmE2YzlkMmVjMy5iaW5kUG9wdXAocG9wdXBfZDk4NWM3ZjM0YjFjNDU0YWE1Y2RlM2VjYzBlODFmYjEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfNTZmMDRhZGI5NDExNDgwMGFlMTk2YTFlOTZiZmQxMmMgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMyNzE4NjQ2Nzg3MjU1MTYsMTAzLjk0NjM0NjEyNzE0MDczXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9kMGVhZGUxNDhiMmQ0MThkYjk3MzQxYTlmM2NjNTNiNCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jYTgyNDJlMjIzYTI0OTY2ODI1ZDRjNTkyOGI1ZWU3YSA9ICQoJzxkaXYgaWQ9Imh0bWxfY2E4MjQyZTIyM2EyNDk2NjgyNWQ0YzU5MjhiNWVlN2EiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlRhbmFoIE1lcmFoIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9kMGVhZGUxNDhiMmQ0MThkYjk3MzQxYTlmM2NjNTNiNC5zZXRDb250ZW50KGh0bWxfY2E4MjQyZTIyM2EyNDk2NjgyNWQ0YzU5MjhiNWVlN2EpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNTZmMDRhZGI5NDExNDgwMGFlMTk2YTFlOTZiZmQxMmMuYmluZFBvcHVwKHBvcHVwX2QwZWFkZTE0OGIyZDQxOGRiOTczNDFhOWYzY2M1M2I0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2ViOTA1MDM4MTI2MTRiZTNiZWRkM2QzNzAyM2JlYjZmID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zODE3NTUzNzk0NDYyMDA1LDEwMy44NDQ5NDY5NTY0MTc4NF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZjVkZDlmMGZiZTY5NDg5MThhYThjNjIwNDg4ZWZiYWMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNTk3OTdlYmU2MDY4NGUwZTgwZTk1YTA3MDM0MDAyYWUgPSAkKCc8ZGl2IGlkPSJodG1sXzU5Nzk3ZWJlNjA2ODRlMGU4MGU5NWEwNzAzNDAwMmFlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5ZaW8gQ2h1IEthbmcgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2Y1ZGQ5ZjBmYmU2OTQ4OTE4YWE4YzYyMDQ4OGVmYmFjLnNldENvbnRlbnQoaHRtbF81OTc5N2ViZTYwNjg0ZTBlODBlOTVhMDcwMzQwMDJhZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9lYjkwNTAzODEyNjE0YmUzYmVkZDNkMzcwMjNiZWI2Zi5iaW5kUG9wdXAocG9wdXBfZjVkZDlmMGZiZTY5NDg5MThhYThjNjIwNDg4ZWZiYWMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfYTVmZjc1Y2U4MTAwNDhiMTk2YzMwOGFjOTQzODk1ZjQgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjQwMjI4NjAxMDgyNzY0OSwxMDMuOTEyNzI3MDIyMjI5MzNdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2YyN2Q2YTc2Mjk4MDQyMzVhODg1MGM1YzQxNjM3ZjNmID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2QxODEyM2M1MGJkMzQzMmVhZWU2ZDIxMGI3MWVlMTFmID0gJCgnPGRpdiBpZD0iaHRtbF9kMTgxMjNjNTBiZDM0MzJlYWVlNmQyMTBiNzFlZTExZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+T2FzaXMgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2YyN2Q2YTc2Mjk4MDQyMzVhODg1MGM1YzQxNjM3ZjNmLnNldENvbnRlbnQoaHRtbF9kMTgxMjNjNTBiZDM0MzJlYWVlNmQyMTBiNzFlZTExZik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9hNWZmNzVjZTgxMDA0OGIxOTZjMzA4YWM5NDM4OTVmNC5iaW5kUG9wdXAocG9wdXBfZjI3ZDZhNzYyOTgwNDIzNWE4ODUwYzVjNDE2MzdmM2YpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMTVlNjQ4MTU3YmNiNDNhMWFlYmNhN2ZhZGZmNDdmNzAgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjQwOTYxMjAxODQyMDQ2NjgsMTAzLjkwNDgzMTIxNzEzMTM5XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yNGU2NTg5ODJjMDU0MWRkOGM4YjAxNDgyODI0M2ExMCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9kOTljODU0MGM4MWE0YzUxODNiOTBkZjI0MGQ2NjY5MiA9ICQoJzxkaXYgaWQ9Imh0bWxfZDk5Yzg1NDBjODFhNGM1MTgzYjkwZGYyNDBkNjY2OTIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNhbSBLZWUgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzI0ZTY1ODk4MmMwNTQxZGQ4YzhiMDE0ODI4MjQzYTEwLnNldENvbnRlbnQoaHRtbF9kOTljODU0MGM4MWE0YzUxODNiOTBkZjI0MGQ2NjY5Mik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl8xNWU2NDgxNTdiY2I0M2ExYWViY2E3ZmFkZmY0N2Y3MC5iaW5kUG9wdXAocG9wdXBfMjRlNjU4OTgyYzA1NDFkZDhjOGIwMTQ4MjgyNDNhMTApOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfY2E2YzE2YmU4MDc3NGQ3N2E0Y2U4Y2EwZjAyMzg3NGEgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM5NzE2OTUyOTc5MTYzMDcsMTAzLjg4OTMwNDQ3NzA3MDk3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yYmRkODEzZDMwOTM0Y2VmODc2N2IzNWQyM2RhZWE4YSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF81MjE2NWM1ZTE3Njg0ZjcyOTc5ZDJhZTc0YjFkNmMyZCA9ICQoJzxkaXYgaWQ9Imh0bWxfNTIxNjVjNWUxNzY4NGY3Mjk3OWQyYWU3NGIxZDZjMmQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkZhcm13YXkgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzJiZGQ4MTNkMzA5MzRjZWY4NzY3YjM1ZDIzZGFlYThhLnNldENvbnRlbnQoaHRtbF81MjE2NWM1ZTE3Njg0ZjcyOTc5ZDJhZTc0YjFkNmMyZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9jYTZjMTZiZTgwNzc0ZDc3YTRjZThjYTBmMDIzODc0YS5iaW5kUG9wdXAocG9wdXBfMmJkZDgxM2QzMDkzNGNlZjg3NjdiMzVkMjNkYWVhOGEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMDg2NjUyYTUxNDNlNDg0MTgyNGZmM2NkYzliNzZjMzkgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM5ODIxMjE2MTczMDY4NDIsMTAzLjg4MTI1NTg4ODYxNzU3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9mNGIxZjhhYjhiZWI0YjcwYTY1MjlkYmUwNjY0Njk0NCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8zMWNmMzQ5ZmM5Yzg0ZGY3YWM2MzY0OGUyMDA4NTBjYyA9ICQoJzxkaXYgaWQ9Imh0bWxfMzFjZjM0OWZjOWM4NGRmN2FjNjM2NDhlMjAwODUwY2MiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkt1cGFuZyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZjRiMWY4YWI4YmViNGI3MGE2NTI5ZGJlMDY2NDY5NDQuc2V0Q29udGVudChodG1sXzMxY2YzNDlmYzljODRkZjdhYzYzNjQ4ZTIwMDg1MGNjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzA4NjY1MmE1MTQzZTQ4NDE4MjRmZjNjZGM5Yjc2YzM5LmJpbmRQb3B1cChwb3B1cF9mNGIxZjhhYjhiZWI0YjcwYTY1MjlkYmUwNjY0Njk0NCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8xMWFhNzUyNzBmNGI0MzQyYWIyZWZjZjE3ZWNjNDczNCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzg2NzIzMjU0ODYxNTQ1NCwxMDMuODkwNTM5MDkzNzg5MjddLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzI2MzQ2Yzg0NmQzNjRkMDFiYzQ1MWQ0MmE5YzkxOWEwID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2U1MmQxZGVmM2I3MjRjY2NhOWRjN2QwY2VhNjdmOTQ3ID0gJCgnPGRpdiBpZD0iaHRtbF9lNTJkMWRlZjNiNzI0Y2NjYTlkYzdkMGNlYTY3Zjk0NyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+UmVuam9uZyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMjYzNDZjODQ2ZDM2NGQwMWJjNDUxZDQyYTljOTE5YTAuc2V0Q29udGVudChodG1sX2U1MmQxZGVmM2I3MjRjY2NhOWRjN2QwY2VhNjdmOTQ3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzExYWE3NTI3MGY0YjQzNDJhYjJlZmNmMTdlY2M0NzM0LmJpbmRQb3B1cChwb3B1cF8yNjM0NmM4NDZkMzY0ZDAxYmM0NTFkNDJhOWM5MTlhMCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8yNmEyNjE0NmQzYTY0ODFjYmEwYjRjZDU0YTA2NmY0MyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzc3OTA5MjMxMzE1ODcxLDEwMy43NjMwMTMzMDA1MDI3M10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNjRjMWIwYTgyOWM2NDMwNDg0OGQ4NGNiODExNzlmYjEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOTMzYjk4ODFiZDAzNGJiZWFmNzBmMDA5ZDliOGI5YzkgPSAkKCc8ZGl2IGlkPSJodG1sXzkzM2I5ODgxYmQwMzRiYmVhZjcwZjAwOWQ5YjhiOWM5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5CdWtpdCBQYW5qYW5nIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF82NGMxYjBhODI5YzY0MzA0ODQ4ZDg0Y2I4MTE3OWZiMS5zZXRDb250ZW50KGh0bWxfOTMzYjk4ODFiZDAzNGJiZWFmNzBmMDA5ZDliOGI5YzkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMjZhMjYxNDZkM2E2NDgxY2JhMGI0Y2Q1NGEwNjZmNDMuYmluZFBvcHVwKHBvcHVwXzY0YzFiMGE4MjljNjQzMDQ4NDhkODRjYjgxMTc5ZmIxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzc0MjhjZjAxZDQ2YjRiMGJiMDJmM2MzZmVhNzFkNTU4ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zODQ1MjAxMjk1MDg4MjEyLDEwMy43NzA4MDgyNDkyMTQ5NV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZGFiZTI5NDZmZmZmNDZiY2E0ZjZmZjk5NzA4ZjVmYzggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZTQyMGJkYTBhZWRmNDlmM2FlZGZhNjNiZDAwZjM3ZDggPSAkKCc8ZGl2IGlkPSJodG1sX2U0MjBiZGEwYWVkZjQ5ZjNhZWRmYTYzYmQwMGYzN2Q4IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5GYWphciBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZGFiZTI5NDZmZmZmNDZiY2E0ZjZmZjk5NzA4ZjVmYzguc2V0Q29udGVudChodG1sX2U0MjBiZGEwYWVkZjQ5ZjNhZWRmYTYzYmQwMGYzN2Q4KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzc0MjhjZjAxZDQ2YjRiMGJiMDJmM2MzZmVhNzFkNTU4LmJpbmRQb3B1cChwb3B1cF9kYWJlMjk0NmZmZmY0NmJjYTRmNmZmOTk3MDhmNWZjOCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl80ZDA3ODA0MDQ5YWU0MmU2ODBiMGRmZGQxYjE5NjJlNCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuNDA1MTk0MDM0NjM2MjMzLDEwMy45MDI0MTE1Nzg2MDE3NF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfODRjYmQ2Yzk1YWRhNDdhYmI0NjJiN2YxYTgxM2UzZjkgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMTUzZGM5YzU5Nzg1NDhlZmI1NTRiNGIyYjUyMTNjODQgPSAkKCc8ZGl2IGlkPSJodG1sXzE1M2RjOWM1OTc4NTQ4ZWZiNTU0YjRiMmI1MjEzYzg0IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5QdW5nZ29sIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF84NGNiZDZjOTVhZGE0N2FiYjQ2MmI3ZjFhODEzZTNmOS5zZXRDb250ZW50KGh0bWxfMTUzZGM5YzU5Nzg1NDhlZmI1NTRiNGIyYjUyMTNjODQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNGQwNzgwNDA0OWFlNDJlNjgwYjBkZmRkMWIxOTYyZTQuYmluZFBvcHVwKHBvcHVwXzg0Y2JkNmM5NWFkYTQ3YWJiNDYyYjdmMWE4MTNlM2Y5KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzFlNWU5OTg4MjA0ZTQzMTU5OTE2MDdhMzJhMTZjMmE3ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS40MTU5MDEwNTI3NzI2MjQsMTAzLjkwMjE1NTk4MzAwNzY1XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9kNmM2YjcyMDczYTA0YmU3ODY2NTFlODhhY2FhODI3YyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jMTQxYmFkNjFjZWI0NjA2OWE2N2U2YjY3ZDM4MGQ2OCA9ICQoJzxkaXYgaWQ9Imh0bWxfYzE0MWJhZDYxY2ViNDYwNjlhNjdlNmI2N2QzODBkNjgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNhbXVkZXJhIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9kNmM2YjcyMDczYTA0YmU3ODY2NTFlODhhY2FhODI3Yy5zZXRDb250ZW50KGh0bWxfYzE0MWJhZDYxY2ViNDYwNjlhNjdlNmI2N2QzODBkNjgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMWU1ZTk5ODgyMDRlNDMxNTk5MTYwN2EzMmExNmMyYTcuYmluZFBvcHVwKHBvcHVwX2Q2YzZiNzIwNzNhMDRiZTc4NjY1MWU4OGFjYWE4MjdjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzNjZWM4NDY0MTc3NjQ3NzI4ZjIyZGZlNTRiMjlmMGFiID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS40MTE4Njk3OTE3ODcwMzU1LDEwMy45MDAzMTM0ODQyOTMxMV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfM2Q2YjBhZDFhOWJiNDU5ZWIzYWM0ZWI5YTA5ZDQwM2IgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfM2ZmNjU1NTlmM2Q2NDE4YWFlYjE4ZTkxYWI4ODQ2ZWMgPSAkKCc8ZGl2IGlkPSJodG1sXzNmZjY1NTU5ZjNkNjQxOGFhZWIxOGU5MWFiODg0NmVjIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5OaWJvbmcgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzNkNmIwYWQxYTliYjQ1OWViM2FjNGViOWEwOWQ0MDNiLnNldENvbnRlbnQoaHRtbF8zZmY2NTU1OWYzZDY0MThhYWViMThlOTFhYjg4NDZlYyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl8zY2VjODQ2NDE3NzY0NzcyOGYyMmRmZTU0YjI5ZjBhYi5iaW5kUG9wdXAocG9wdXBfM2Q2YjBhZDFhOWJiNDU5ZWIzYWM0ZWI5YTA5ZDQwM2IpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfNzIzMzU5NjU1YmZjNDg0ZGFlYWVmOTEwMWNhOTEwN2UgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjQwNTA4NzkxODE4NzY5MzIsMTAzLjg5NzIwOTMxNjk0MjI1XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9kZmJkZDM4ODllNzk0MWY0ODEwY2NmNjgxZDkzNzViZiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jNjlhNGExNWMyNGM0Y2JjODA5Y2U4MDc4Y2NlOWE5ZCA9ICQoJzxkaXYgaWQ9Imh0bWxfYzY5YTRhMTVjMjRjNGNiYzgwOWNlODA3OGNjZTlhOWQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNvbyBUZWNrIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9kZmJkZDM4ODllNzk0MWY0ODEwY2NmNjgxZDkzNzViZi5zZXRDb250ZW50KGh0bWxfYzY5YTRhMTVjMjRjNGNiYzgwOWNlODA3OGNjZTlhOWQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNzIzMzU5NjU1YmZjNDg0ZGFlYWVmOTEwMWNhOTEwN2UuYmluZFBvcHVwKHBvcHVwX2RmYmRkMzg4OWU3OTQxZjQ4MTBjY2Y2ODFkOTM3NWJmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2YzNjY2MDQ5NDVkNDRmMDRiNjU5ODg5MmE1M2JlNTAzID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zOTk1ODQxODYwOTUwOTc1LDEwMy45MTY0ODYzMjg1NjQyMl0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNWViNWNiYjI1MjlmNGFhYTg2NjhlOWU2ZDJlMTkwZmEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNTgzZWVkZDdiNDQ3NGFlZGI2M2Y3YTU1ZjE0MWVjOTMgPSAkKCc8ZGl2IGlkPSJodG1sXzU4M2VlZGQ3YjQ0NzRhZWRiNjNmN2E1NWYxNDFlYzkzIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5LYWRhbG9vciBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNWViNWNiYjI1MjlmNGFhYTg2NjhlOWU2ZDJlMTkwZmEuc2V0Q29udGVudChodG1sXzU4M2VlZGQ3YjQ0NzRhZWRiNjNmN2E1NWYxNDFlYzkzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2YzNjY2MDQ5NDVkNDRmMDRiNjU5ODg5MmE1M2JlNTAzLmJpbmRQb3B1cChwb3B1cF81ZWI1Y2JiMjUyOWY0YWFhODY2OGU5ZTZkMmUxOTBmYSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8zZjA2YThmZjQzZjg0NjRjYjk1ZTlmOGU5ZmI0NTM3OCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzkzOTA4NTU5NDUxMzc3OCwxMDMuOTEyNTgwNTIxNDA4ODRdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzZiNjE3ZDkxYjkzYTRmODE4Zjc4NTY4ZjI0ODRmYTRlID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzIwZDI5OWIwODAyZDQzYjk5YWMwZjZiNWU0MGE0ODlmID0gJCgnPGRpdiBpZD0iaHRtbF8yMGQyOTliMDgwMmQ0M2I5OWFjMGY2YjVlNDBhNDg5ZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+Q29yYWwgRWRnZSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNmI2MTdkOTFiOTNhNGY4MThmNzg1NjhmMjQ4NGZhNGUuc2V0Q29udGVudChodG1sXzIwZDI5OWIwODAyZDQzYjk5YWMwZjZiNWU0MGE0ODlmKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzNmMDZhOGZmNDNmODQ2NGNiOTVlOWY4ZTlmYjQ1Mzc4LmJpbmRQb3B1cChwb3B1cF82YjYxN2Q5MWI5M2E0ZjgxOGY3ODU2OGYyNDg0ZmE0ZSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8zZjBhMGMwZWJmMDI0YzQ5YTdjMTc0ZDRiMjMxZDE0ZiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzk3MzE3NDg4MzI4Njc0LDEwMy44NzU2MzQ4MjMxMDM4N10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZTA1MGM2OGQ1NzY1NGQ2MWIzYTZlYTQ3ZTgzZDkzZGQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNjU0NmE1ODZlZjE2NGE5NGEzODk2YmM2YjkyMWI5ZmQgPSAkKCc8ZGl2IGlkPSJodG1sXzY1NDZhNTg2ZWYxNjRhOTRhMzg5NmJjNmI5MjFiOWZkIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5UaGFuZ2dhbSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZTA1MGM2OGQ1NzY1NGQ2MWIzYTZlYTQ3ZTgzZDkzZGQuc2V0Q29udGVudChodG1sXzY1NDZhNTg2ZWYxNjRhOTRhMzg5NmJjNmI5MjFiOWZkKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzNmMGEwYzBlYmYwMjRjNDlhN2MxNzRkNGIyMzFkMTRmLmJpbmRQb3B1cChwb3B1cF9lMDUwYzY4ZDU3NjU0ZDYxYjNhNmVhNDdlODNkOTNkZCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl80MDIzNmUwOTEwMjM0MTFkOWZkMDAzZDE0YjViNDUxNSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzc4NjAyMDk5MzE2OTE4LDEwMy43NDkwNTUxODYxNDI4OF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZGY0NDA1YTM3YTM5NDMzMGEzZDg2MjcxNGRhZjJiZDIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMjhjMGQ5ZjgwYTgzNDg2OGE4ZGIyMTgzNjJkNDc0MWIgPSAkKCc8ZGl2IGlkPSJodG1sXzI4YzBkOWY4MGE4MzQ4NjhhOGRiMjE4MzYyZDQ3NDFiIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5LZWF0IEhvbmcgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2RmNDQwNWEzN2EzOTQzMzBhM2Q4NjI3MTRkYWYyYmQyLnNldENvbnRlbnQoaHRtbF8yOGMwZDlmODBhODM0ODY4YThkYjIxODM2MmQ0NzQxYik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl80MDIzNmUwOTEwMjM0MTFkOWZkMDAzZDE0YjViNDUxNS5iaW5kUG9wdXAocG9wdXBfZGY0NDA1YTM3YTM5NDMzMGEzZDg2MjcxNGRhZjJiZDIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfM2I1ZmFkZjIyYzllNDk4YzlmZjA5YTJmNjUxY2RhMGIgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjQwNTIzNDE2OTUwNTg4MjUsMTAzLjkwODYwMzIxMDI1MDYyXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yN2NhYjdkZWQyZDY0NzQwYmZiNGM5MTc2ZjA2YTJkNSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8yYjM3MWNmNzBmYjI0MTAwODljYTkzMTliYmRiY2M4ZCA9ICQoJzxkaXYgaWQ9Imh0bWxfMmIzNzFjZjcwZmIyNDEwMDg5Y2E5MzE5YmJkYmNjOGQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkRhbWFpIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yN2NhYjdkZWQyZDY0NzQwYmZiNGM5MTc2ZjA2YTJkNS5zZXRDb250ZW50KGh0bWxfMmIzNzFjZjcwZmIyNDEwMDg5Y2E5MzE5YmJkYmNjOGQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfM2I1ZmFkZjIyYzllNDk4YzlmZjA5YTJmNjUxY2RhMGIuYmluZFBvcHVwKHBvcHVwXzI3Y2FiN2RlZDJkNjQ3NDBiZmI0YzkxNzZmMDZhMmQ1KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2U2YjJiOWMwOWYzMTRmY2M4NWUxNjFmM2FiOTYwMWFlID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zOTkyODEzMTgzNTM3NTExLDEwMy45MDU5NjE2MTA2ODc4N10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMmY2NDZhOWMxOWE2NGY4Nzk0YmY3NGU1Njc0NDBlNGMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMjk3M2M3YTgwMGU5NGU5ODk4NzVlMjFkZmY1YjNkYTEgPSAkKCc8ZGl2IGlkPSJodG1sXzI5NzNjN2E4MDBlOTRlOTg5ODc1ZTIxZGZmNWIzZGExIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5Db3ZlIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yZjY0NmE5YzE5YTY0Zjg3OTRiZjc0ZTU2NzQ0MGU0Yy5zZXRDb250ZW50KGh0bWxfMjk3M2M3YTgwMGU5NGU5ODk4NzVlMjFkZmY1YjNkYTEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZTZiMmI5YzA5ZjMxNGZjYzg1ZTE2MWYzYWI5NjAxYWUuYmluZFBvcHVwKHBvcHVwXzJmNjQ2YTljMTlhNjRmODc5NGJmNzRlNTY3NDQwZTRjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2QxNjgyNzdiMzI1ZjRiMjdhZmNkYzFkZDg5OGQ1ZjhmID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zOTQ1MjM4MjkxMDI5NjczLDEwMy45MTYxNjU3MjU1Nzk3M10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYzQwY2U4M2U2MTQ1NGExODgwYWUzMmM0MDFhMTVjOGQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNmRhN2IxZjcwYzVkNGNhMzk0Y2NlZWU1NzVjMTQwNjUgPSAkKCc8ZGl2IGlkPSJodG1sXzZkYTdiMWY3MGM1ZDRjYTM5NGNjZWVlNTc1YzE0MDY1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5SaXZpZXJhIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9jNDBjZTgzZTYxNDU0YTE4ODBhZTMyYzQwMWExNWM4ZC5zZXRDb250ZW50KGh0bWxfNmRhN2IxZjcwYzVkNGNhMzk0Y2NlZWU1NzVjMTQwNjUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZDE2ODI3N2IzMjVmNGIyN2FmY2RjMWRkODk4ZDVmOGYuYmluZFBvcHVwKHBvcHVwX2M0MGNlODNlNjE0NTRhMTg4MGFlMzJjNDAxYTE1YzhkKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzFiMGI1MTU1NDQzZjRhMGY4ZTdiOTM2NzY0YzViYjU4ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zODI2OTE2MjkxNTc4NjQ4LDEwMy43NjIzNjY4OTI2NjkxNl0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZjc2ZWVkMWViNWNjNGJlM2E5YTFkYzMyMjVmZTJmNGEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOGFkYWQ3YWI0NzExNDViOWIyY2VmNTAzNmI0Y2U1MzUgPSAkKCc8ZGl2IGlkPSJodG1sXzhhZGFkN2FiNDcxMTQ1YjliMmNlZjUwMzZiNGNlNTM1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TZW5qYSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZjc2ZWVkMWViNWNjNGJlM2E5YTFkYzMyMjVmZTJmNGEuc2V0Q29udGVudChodG1sXzhhZGFkN2FiNDcxMTQ1YjliMmNlZjUwMzZiNGNlNTM1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzFiMGI1MTU1NDQzZjRhMGY4ZTdiOTM2NzY0YzViYjU4LmJpbmRQb3B1cChwb3B1cF9mNzZlZWQxZWI1Y2M0YmUzYTlhMWRjMzIyNWZlMmY0YSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl83M2VhN2Q5MTJjNWM0ZDcyYTU5M2IyNWQ1Y2Q4N2UyMCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzg2NzAyMzU4MzQ3NjE1NSwxMDMuNzY0NTAzMDA0ODc0MzZdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzg4M2U1Yzk3YWRkNzQ1OTI4ZjU1YzUwMWVhZWVmMDI2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzU5NjE2ODJhYjJjZDQwYjZiZDFlMjc3MjVjMDNlZjYyID0gJCgnPGRpdiBpZD0iaHRtbF81OTYxNjgyYWIyY2Q0MGI2YmQxZTI3NzI1YzAzZWY2MiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+SmVsYXBhbmcgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzg4M2U1Yzk3YWRkNzQ1OTI4ZjU1YzUwMWVhZWVmMDI2LnNldENvbnRlbnQoaHRtbF81OTYxNjgyYWIyY2Q0MGI2YmQxZTI3NzI1YzAzZWY2Mik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl83M2VhN2Q5MTJjNWM0ZDcyYTU5M2IyNWQ1Y2Q4N2UyMC5iaW5kUG9wdXAocG9wdXBfODgzZTVjOTdhZGQ3NDU5MjhmNTVjNTAxZWFlZWYwMjYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZDU0OWM0MmUwYTE2NDU3ZGE1OWVjMmQ3NjU3M2E1NTIgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM3Nzc0OTY2NzE1MDgzNzQsMTAzLjc2NjY2ODYyNjkzNDc5XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85MDQ2ZWRiZWI3NTU0MGI2ODdjYzNiZTNmOWQ0NDY2ZiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85ZmU5NTMzZTliZWM0ODJhYjY1MjI5ZWQ1YTNjNDc3MiA9ICQoJzxkaXYgaWQ9Imh0bWxfOWZlOTUzM2U5YmVjNDgyYWI2NTIyOWVkNWEzYzQ3NzIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlBldGlyIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF85MDQ2ZWRiZWI3NTU0MGI2ODdjYzNiZTNmOWQ0NDY2Zi5zZXRDb250ZW50KGh0bWxfOWZlOTUzM2U5YmVjNDgyYWI2NTIyOWVkNWEzYzQ3NzIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZDU0OWM0MmUwYTE2NDU3ZGE1OWVjMmQ3NjU3M2E1NTIuYmluZFBvcHVwKHBvcHVwXzkwNDZlZGJlYjc1NTQwYjY4N2NjM2JlM2Y5ZDQ0NjZmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzEyNTc2MjJlZmVjYzQ3Y2VhOGY1NzUxZjZiOWNlYTY5ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNzY2ODQwMTI3Nzc3MTEsMTAzLjc1MzcxMTg5ODkzMzIzXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yNmY5ZTEyZjJlNjI0Y2Q3ODJmZDNkOTllMTI1NjgxZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80MDQ4YTVkZTI5YWE0NTdkYmIwMmE0ZWU0MzllNDg0MyA9ICQoJzxkaXYgaWQ9Imh0bWxfNDA0OGE1ZGUyOWFhNDU3ZGJiMDJhNGVlNDM5ZTQ4NDMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlRlY2sgV2h5ZSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMjZmOWUxMmYyZTYyNGNkNzgyZmQzZDk5ZTEyNTY4MWUuc2V0Q29udGVudChodG1sXzQwNDhhNWRlMjlhYTQ1N2RiYjAyYTRlZTQzOWU0ODQzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzEyNTc2MjJlZmVjYzQ3Y2VhOGY1NzUxZjZiOWNlYTY5LmJpbmRQb3B1cChwb3B1cF8yNmY5ZTEyZjJlNjI0Y2Q3ODJmZDNkOTllMTI1NjgxZSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8xODM4ODlmNGMwMTY0MzU1YTQzNjdlMmMwMjE3NTY4NSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzgwMjk3NjIwNTUxMjQ5OCwxMDMuNzQ1MjkxNDY2Nzk0NTRdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzYxMzQyZmM0MTAxMjRkZmVhZDdkMWMwY2M3MzhjMTc2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2JkNjc1OTE3NGJlNjQzNzU4YWNlNDZmMWVhYjRkOGVjID0gJCgnPGRpdiBpZD0iaHRtbF9iZDY3NTkxNzRiZTY0Mzc1OGFjZTQ2ZjFlYWI0ZDhlYyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U291dGggVmlldyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNjEzNDJmYzQxMDEyNGRmZWFkN2QxYzBjYzczOGMxNzYuc2V0Q29udGVudChodG1sX2JkNjc1OTE3NGJlNjQzNzU4YWNlNDZmMWVhYjRkOGVjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzE4Mzg4OWY0YzAxNjQzNTVhNDM2N2UyYzAyMTc1Njg1LmJpbmRQb3B1cChwb3B1cF82MTM0MmZjNDEwMTI0ZGZlYWQ3ZDFjMGNjNzM4YzE3Nik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8zYzNlOTEzYjUyZDI0NTdmOTIwMWZkZTU3MTgxODUwZCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzkyMDc5MTcyMTA2MDYyMSwxMDMuODgwMDI5MjY2OTk0M10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMjg5MDg2ZTEyMzIyNDBmY2I0NWE2ODgzNzY2MTNlNDggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNTg1M2QzMDBlOGVjNGE1ZWFiOWZhNThjNWY0OTZhMGYgPSAkKCc8ZGl2IGlkPSJodG1sXzU4NTNkMzAwZThlYzRhNWVhYjlmYTU4YzVmNDk2YTBmIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5MYXlhciBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMjg5MDg2ZTEyMzIyNDBmY2I0NWE2ODgzNzY2MTNlNDguc2V0Q29udGVudChodG1sXzU4NTNkMzAwZThlYzRhNWVhYjlmYTU4YzVmNDk2YTBmKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzNjM2U5MTNiNTJkMjQ1N2Y5MjAxZmRlNTcxODE4NTBkLmJpbmRQb3B1cChwb3B1cF8yODkwODZlMTIzMjI0MGZjYjQ1YTY4ODM3NjYxM2U0OCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl80MjVkZWRmMTFiOGU0ZDY3ODhmOTE0OTE4NWQ1YmJhYiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzgzOTc4MjM1MTY3MjE4NSwxMDMuOTAyMTczNjU1Mzg0MzZdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2ZjNjVlYWMxOTE0OTQwMDFiOTZjOGMwMTJhMDE2OWVkID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2IzZjUwNWQyZjM4ZDQxODFhNjRmOGM3NDk5NTk0Yzg5ID0gJCgnPGRpdiBpZD0iaHRtbF9iM2Y1MDVkMmYzOGQ0MTgxYTY0ZjhjNzQ5OTU5NGM4OSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+S2FuZ2thciBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZmM2NWVhYzE5MTQ5NDAwMWI5NmM4YzAxMmEwMTY5ZWQuc2V0Q29udGVudChodG1sX2IzZjUwNWQyZjM4ZDQxODFhNjRmOGM3NDk5NTk0Yzg5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzQyNWRlZGYxMWI4ZTRkNjc4OGY5MTQ5MTg1ZDViYmFiLmJpbmRQb3B1cChwb3B1cF9mYzY1ZWFjMTkxNDk0MDAxYjk2YzhjMDEyYTAxNjllZCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl82NmNiNDBlOTk5YWU0MmEyYTI3NWRjZThkOWFlZTRjOSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzk2OTExMzg2Mzk3MTI0NSwxMDMuOTA4OTQ5ODgzNTcyNDhdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2RhNjNkOGMwZjk3NDQ2NTM4NzY5M2NmOWQ1YzhlYTE3ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzI5ZGNmMWVhYzhmZjQ0YzI4YjhlZWJiNDI5NmZlZDBjID0gJCgnPGRpdiBpZD0iaHRtbF8yOWRjZjFlYWM4ZmY0NGMyOGI4ZWViYjQyOTZmZWQwYyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+TWVyaWRpYW4gTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2RhNjNkOGMwZjk3NDQ2NTM4NzY5M2NmOWQ1YzhlYTE3LnNldENvbnRlbnQoaHRtbF8yOWRjZjFlYWM4ZmY0NGMyOGI4ZWViYjQyOTZmZWQwYyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl82NmNiNDBlOTk5YWU0MmEyYTI3NWRjZThkOWFlZTRjOS5iaW5kUG9wdXAocG9wdXBfZGE2M2Q4YzBmOTc0NDY1Mzg3NjkzY2Y5ZDVjOGVhMTcpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfNjJkMDFmNDAzYjFmNDcwNjg2YThmODZmZWEzNzNhYzIgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM5NjI3Njk2NDY2OTQ0NTUsMTAzLjg5Mzc5Njg0NzQ2NTkxXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xODI4MzBkMzQxYTQ0ZGMyYTJiY2Q5NDgwYzQ1YmE0MSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8wODJlMzFhNTI4ZmI0MDFjYjY5MjVmZjg3Mjk1YTU0YSA9ICQoJzxkaXYgaWQ9Imh0bWxfMDgyZTMxYTUyOGZiNDAxY2I2OTI1ZmY4NzI5NWE1NGEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkNoZW5nIExpbSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMTgyODMwZDM0MWE0NGRjMmEyYmNkOTQ4MGM0NWJhNDEuc2V0Q29udGVudChodG1sXzA4MmUzMWE1MjhmYjQwMWNiNjkyNWZmODcyOTVhNTRhKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzYyZDAxZjQwM2IxZjQ3MDY4NmE4Zjg2ZmVhMzczYWMyLmJpbmRQb3B1cChwb3B1cF8xODI4MzBkMzQxYTQ0ZGMyYTJiY2Q5NDgwYzQ1YmE0MSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl84ZGM0MmZlNmEyZWY0YTgyOTg1ZmNiNDAzM2VmMWQ1MSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuNDE2ODQ3ODUzODI5Njk0NywxMDMuOTA2NjUwNDU3MDc1MjZdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzAyMzQ5NTdkYTdkMjQ3MzVhZmE0YjI5ODBlNDFjOTIzID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzgyODg4OTUzMGM0YzQzNzA5MWI2NTNjN2VmYzM2M2YzID0gJCgnPGRpdiBpZD0iaHRtbF84Mjg4ODk1MzBjNGM0MzcwOTFiNjUzYzdlZmMzNjNmMyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+UHVuZ2dvbCBQb2ludCBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMDIzNDk1N2RhN2QyNDczNWFmYTRiMjk4MGU0MWM5MjMuc2V0Q29udGVudChodG1sXzgyODg4OTUzMGM0YzQzNzA5MWI2NTNjN2VmYzM2M2YzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzhkYzQyZmU2YTJlZjRhODI5ODVmY2I0MDMzZWYxZDUxLmJpbmRQb3B1cChwb3B1cF8wMjM0OTU3ZGE3ZDI0NzM1YWZhNGIyOTgwZTQxYzkyMyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9kMGYzZDhjYmE2NzA0MzM0YmQyZDQxYmVmNDFlNWI4MyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuNDA4NDUxNzU4OTgxNDYxLDEwMy44OTg1NTgxMjA2NzI2Ml0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMjBlOTIzZWNlMGEyNDNjZDllOTcxNDA5ZjYyNmU1YWQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZDkyMzYzNTU4OTdhNGUxMmE5OTYwOWExMjQ4ZmUwYmQgPSAkKCc8ZGl2IGlkPSJodG1sX2Q5MjM2MzU1ODk3YTRlMTJhOTk2MDlhMTI0OGZlMGJkIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TdW1hbmcgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzIwZTkyM2VjZTBhMjQzY2Q5ZTk3MTQwOWY2MjZlNWFkLnNldENvbnRlbnQoaHRtbF9kOTIzNjM1NTg5N2E0ZTEyYTk5NjA5YTEyNDhmZTBiZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9kMGYzZDhjYmE2NzA0MzM0YmQyZDQxYmVmNDFlNWI4My5iaW5kUG9wdXAocG9wdXBfMjBlOTIzZWNlMGEyNDNjZDllOTcxNDA5ZjYyNmU1YWQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMjQwNGEzMWNmOTE1NDM0YWJiMzlhYWNkM2MzNzRkZTcgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM5MTg4NTIyMTA3NzUyOTQsMTAzLjg3NjMwODI3ODAxNjE3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81NzMxZTdlMTFiY2I0ZjRkYjhmOTE5ODc5ZGRlYzUzNyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jNGUwYzBmZDUyYjU0NDE3YmY4N2I1NWNmNTA0NTdmZSA9ICQoJzxkaXYgaWQ9Imh0bWxfYzRlMGMwZmQ1MmI1NDQxN2JmODdiNTVjZjUwNDU3ZmUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkZlcm52YWxlIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF81NzMxZTdlMTFiY2I0ZjRkYjhmOTE5ODc5ZGRlYzUzNy5zZXRDb250ZW50KGh0bWxfYzRlMGMwZmQ1MmI1NDQxN2JmODdiNTVjZjUwNDU3ZmUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMjQwNGEzMWNmOTE1NDM0YWJiMzlhYWNkM2MzNzRkZTcuYmluZFBvcHVwKHBvcHVwXzU3MzFlN2UxMWJjYjRmNGRiOGY5MTk4NzlkZGVjNTM3KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2U1OTVkMmM0ODRkNzRkYTViYjNhYzI2YzQ4ZTc4ZjhlID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zODkzNDcyODYyOTMyMDEzLDEwMy44ODU4NDM4MTY1NDY5XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iZmE1MTA4MDJhMTY0MWViYmVmYmI2ZjZhYzZkNDUyMiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9lYTNiNzU0OThkN2Q0NjE0OGFjYjM5NGUxY2M1YTFmNiA9ICQoJzxkaXYgaWQ9Imh0bWxfZWEzYjc1NDk4ZDdkNDYxNDhhY2IzOTRlMWNjNWExZjYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlRvbmdrYW5nIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iZmE1MTA4MDJhMTY0MWViYmVmYmI2ZjZhYzZkNDUyMi5zZXRDb250ZW50KGh0bWxfZWEzYjc1NDk4ZDdkNDYxNDhhY2IzOTRlMWNjNWExZjYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZTU5NWQyYzQ4NGQ3NGRhNWJiM2FjMjZjNDhlNzhmOGUuYmluZFBvcHVwKHBvcHVwX2JmYTUxMDgwMmExNjQxZWJiZWZiYjZmNmFjNmQ0NTIyKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2JkYTVhMjU4M2E3ZDRkNGE4ZmUyMjBiZjJlY2QwZjQ5ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNzYxNDIyMTYxMTg1OTQzLDEwMy43NzEyNjk4NDUzMTc2Ml0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDY0NTVhN2IwY2I1NGIyMTk2OWNiZWYwMzFjNGJiOWMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNmI2MDRiNjg2OGNjNGQ5N2I2Mzc3MWFkM2FiNjAzOGUgPSAkKCc8ZGl2IGlkPSJodG1sXzZiNjA0YjY4NjhjYzRkOTdiNjM3NzFhZDNhYjYwMzhlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5QZW5kaW5nIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80NjQ1NWE3YjBjYjU0YjIxOTY5Y2JlZjAzMWM0YmI5Yy5zZXRDb250ZW50KGh0bWxfNmI2MDRiNjg2OGNjNGQ5N2I2Mzc3MWFkM2FiNjAzOGUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfYmRhNWEyNTgzYTdkNGQ0YThmZTIyMGJmMmVjZDBmNDkuYmluZFBvcHVwKHBvcHVwXzQ2NDU1YTdiMGNiNTRiMjE5NjljYmVmMDMxYzRiYjljKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzE2ZTRhNTdjNzU0MTQzZjRhNThjZjdiZTliZjllNjBhID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS40MTI3NzAyMjcxOTc0NTA3LDEwMy45MDY1NzcyODkzMTEzOV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDExNDY5YzIzY2IxNGJmYmFmZDg4YjVmOGEzZDY1MzUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZmRiMjgyZDg5OGM4NGUzZmFjMWE1YWZhM2RlYWU0MjMgPSAkKCc8ZGl2IGlkPSJodG1sX2ZkYjI4MmQ4OThjODRlM2ZhYzFhNWFmYTNkZWFlNDIzIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5UZWNrIExlZSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNDExNDY5YzIzY2IxNGJmYmFmZDg4YjVmOGEzZDY1MzUuc2V0Q29udGVudChodG1sX2ZkYjI4MmQ4OThjODRlM2ZhYzFhNWFmYTNkZWFlNDIzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzE2ZTRhNTdjNzU0MTQzZjRhNThjZjdiZTliZjllNjBhLmJpbmRQb3B1cChwb3B1cF80MTE0NjljMjNjYjE0YmZiYWZkODhiNWY4YTNkNjUzNSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8yMGZlMTM5ZGVmZDE0MWIzOTNiY2EyYWQyYTAxMDU5MCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzgwMDE3MjMwNTc1NjQyNiwxMDMuNzcyNjQ4NzM5Mjc0OTNdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2ExNDg2MTZhNWQzMzQ1Y2I4ZmQ2MzdkMWY3NDViMjAwID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzRiNDg4M2RlM2RhNzQxYjI4MzdiMzljMDY4YTBjOGI3ID0gJCgnPGRpdiBpZD0iaHRtbF80YjQ4ODNkZTNkYTc0MWIyODM3YjM5YzA2OGEwYzhiNyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+QmFuZ2tpdCBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYTE0ODYxNmE1ZDMzNDVjYjhmZDYzN2QxZjc0NWIyMDAuc2V0Q29udGVudChodG1sXzRiNDg4M2RlM2RhNzQxYjI4MzdiMzljMDY4YTBjOGI3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzIwZmUxMzlkZWZkMTQxYjM5M2JjYTJhZDJhMDEwNTkwLmJpbmRQb3B1cChwb3B1cF9hMTQ4NjE2YTVkMzM0NWNiOGZkNjM3ZDFmNzQ1YjIwMCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl83ZjgwMzA3ODNmYzc0MDUxOTBmMDEyZDAyODhiYjI0ZCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzg4MDkyMDM3MTA4ODU5LDEwMy45MDU0Mzg3NzA4NTMxNV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOTU0NWI4ZmFhNDhmNDZjZmJmNzk0ODNjOTVmNDUwZTUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNGM5N2E5ZWIxNTc1NGNjODk1YmZkMjY1YjJlNTFkZjUgPSAkKCc8ZGl2IGlkPSJodG1sXzRjOTdhOWViMTU3NTRjYzg5NWJmZDI2NWIyZTUxZGY1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5CYWthdSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOTU0NWI4ZmFhNDhmNDZjZmJmNzk0ODNjOTVmNDUwZTUuc2V0Q29udGVudChodG1sXzRjOTdhOWViMTU3NTRjYzg5NWJmZDI2NWIyZTUxZGY1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzdmODAzMDc4M2ZjNzQwNTE5MGYwMTJkMDI4OGJiMjRkLmJpbmRQb3B1cChwb3B1cF85NTQ1YjhmYWE0OGY0NmNmYmY3OTQ4M2M5NWY0NTBlNSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9mMmY3OGIwNWZlYzI0MDgzODkyMTM4ZjljYTU3YjEzZCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzkxNDY3ODMwMTcyMzMxMywxMDMuOTA1OTczMjYxOTIyODFdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzUwODZiNzI4NjQ5ZDQ0MGM5OTI4NWRjNzc1ODIxNjk4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2MxZDkzNTVkNzgzYjQ5ZjdiNDM3MzJkYWIzYWJhYzhkID0gJCgnPGRpdiBpZD0iaHRtbF9jMWQ5MzU1ZDc4M2I0OWY3YjQzNzMyZGFiM2FiYWM4ZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+UnVtYmlhIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF81MDg2YjcyODY0OWQ0NDBjOTkyODVkYzc3NTgyMTY5OC5zZXRDb250ZW50KGh0bWxfYzFkOTM1NWQ3ODNiNDlmN2I0MzczMmRhYjNhYmFjOGQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZjJmNzhiMDVmZWMyNDA4Mzg5MjEzOGY5Y2E1N2IxM2QuYmluZFBvcHVwKHBvcHVwXzUwODZiNzI4NjQ5ZDQ0MGM5OTI4NWRjNzc1ODIxNjk4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2YwMzM2Njk1OWRiMzQxYThiMzVjNGIxNzVjYTRiYjdmID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zOTQ0OTIzNzk1MzE3NzgsMTAzLjkwMDQ5MjExNzI0ODIyXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8wNzZkMDJkNGQyMTA0NTA4YmEzNjJmNDI3YTg3OTM4ZCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9lZDZjMTY4NDE4YjA0ODc3ODI1ZDJhNjc1ODNkMTNhZSA9ICQoJzxkaXYgaWQ9Imh0bWxfZWQ2YzE2ODQxOGIwNDg3NzgyNWQyYTY3NTgzZDEzYWUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkNvbXBhc3N2YWxlIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wNzZkMDJkNGQyMTA0NTA4YmEzNjJmNDI3YTg3OTM4ZC5zZXRDb250ZW50KGh0bWxfZWQ2YzE2ODQxOGIwNDg3NzgyNWQyYTY3NTgzZDEzYWUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZjAzMzY2OTU5ZGIzNDFhOGIzNWM0YjE3NWNhNGJiN2YuYmluZFBvcHVwKHBvcHVwXzA3NmQwMmQ0ZDIxMDQ1MDhiYTM2MmY0MjdhODc5MzhkKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2MwYzEyOWM0MDQ1ZDQ3ZDNhNWM4NGI5NWU0M2Q5YjdmID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zODAzMjAzNDE4MjY4MjM3LDEwMy43NjAxMzk0NDQwNzgwNV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfODMwNWM2OTNhZWJhNDM1Y2IxODRjMjUzM2RkYzk3MTUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZmFkYmExYTFhYzM0NDkzNTg0OTllZmMyNzc4OWUxYWMgPSAkKCc8ZGl2IGlkPSJodG1sX2ZhZGJhMWExYWMzNDQ5MzU4NDk5ZWZjMjc3ODllMWFjIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5UZW4gTWlsZSBKdW5jdGlvbiBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfODMwNWM2OTNhZWJhNDM1Y2IxODRjMjUzM2RkYzk3MTUuc2V0Q29udGVudChodG1sX2ZhZGJhMWExYWMzNDQ5MzU4NDk5ZWZjMjc3ODllMWFjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2MwYzEyOWM0MDQ1ZDQ3ZDNhNWM4NGI5NWU0M2Q5YjdmLmJpbmRQb3B1cChwb3B1cF84MzA1YzY5M2FlYmE0MzVjYjE4NGMyNTMzZGRjOTcxNSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9jMTY1NTBkYTRiZmY0YzBmYWI0N2FjOGRkYTBmZjZkYyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzg0MjMyODk0MDA0MTk2LDEwMy44OTcxOTQzMzYxMzg2MV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYWVmOGIxYTZlNzgyNGY2YWIyNjMzMjhhNmRiMmE0NzggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMjgxNjg4MjUwOGVkNDExN2I1YjE3NzQ3Mzk5YWJkZTYgPSAkKCc8ZGl2IGlkPSJodG1sXzI4MTY4ODI1MDhlZDQxMTdiNWIxNzc0NzM5OWFiZGU2IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5SYW5nZ3VuZyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYWVmOGIxYTZlNzgyNGY2YWIyNjMzMjhhNmRiMmE0Nzguc2V0Q29udGVudChodG1sXzI4MTY4ODI1MDhlZDQxMTdiNWIxNzc0NzM5OWFiZGU2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2MxNjU1MGRhNGJmZjRjMGZhYjQ3YWM4ZGRhMGZmNmRjLmJpbmRQb3B1cChwb3B1cF9hZWY4YjFhNmU3ODI0ZjZhYjI2MzMyOGE2ZGIyYTQ3OCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl80NjI5ZDQwZTZiMTI0MDc5YjFjY2U5Y2Y2ZWIwYzUxOSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzc4NjE4MTc3Nzg2NjQ2NCwxMDMuNzU4MDMzNzY1MTEyM10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMzQyN2EyNTkxNDE4NDI2ZWJlM2ViZjUwZjUzMDFlZmUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYmI2MTc5NzA5NTAzNDQ4MTg2YTNhYWQwZjViYjBhMjkgPSAkKCc8ZGl2IGlkPSJodG1sX2JiNjE3OTcwOTUwMzQ0ODE4NmEzYWFkMGY1YmIwYTI5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5QaG9lbml4IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8zNDI3YTI1OTE0MTg0MjZlYmUzZWJmNTBmNTMwMWVmZS5zZXRDb250ZW50KGh0bWxfYmI2MTc5NzA5NTAzNDQ4MTg2YTNhYWQwZjViYjBhMjkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNDYyOWQ0MGU2YjEyNDA3OWIxY2NlOWNmNmViMGM1MTkuYmluZFBvcHVwKHBvcHVwXzM0MjdhMjU5MTQxODQyNmViZTNlYmY1MGY1MzAxZWZlKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzE1N2UzZjMwOTI5ZDQzOGM4NTM2MGNlNDQwZTEwMDMwID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zODc3NzE0NjQxOTQ1MDU2LDEwMy43Njk1OTgwNjM0ODgzN10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDk5ODI3YTMwNTI0NDE3ODk1YmNkZWY1ZDQ2MjI5MzcgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMmQ1YzA4YWEzZjliNGNlOTgyMjA1ZWM0MDgxMzEwZTMgPSAkKCc8ZGl2IGlkPSJodG1sXzJkNWMwOGFhM2Y5YjRjZTk4MjIwNWVjNDA4MTMxMGUzIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TZWdhciBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNDk5ODI3YTMwNTI0NDE3ODk1YmNkZWY1ZDQ2MjI5Mzcuc2V0Q29udGVudChodG1sXzJkNWMwOGFhM2Y5YjRjZTk4MjIwNWVjNDA4MTMxMGUzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzE1N2UzZjMwOTI5ZDQzOGM4NTM2MGNlNDQwZTEwMDMwLmJpbmRQb3B1cChwb3B1cF80OTk4MjdhMzA1MjQ0MTc4OTViY2RlZjVkNDYyMjkzNyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9jYTgyZjk2MjY3NTI0ZjhlYWQ4MzJiYzQ4MjEzNTY3NCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzkxNjA4Njk3MTQ5ODk3LDEwMy44OTU0NDIyNTAyOTIyNl0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNzMzNjZmOGYyYWY3NDQyN2I5MzYyZDkxNzQxYWQ3NmYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYjJlNzY0NWI2NDQ1NDI5Y2FhMmUyOTAyMTZlMThhYjIgPSAkKCc8ZGl2IGlkPSJodG1sX2IyZTc2NDViNjQ0NTQyOWNhYTJlMjkwMjE2ZTE4YWIyIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TZW5na2FuZyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNzMzNjZmOGYyYWY3NDQyN2I5MzYyZDkxNzQxYWQ3NmYuc2V0Q29udGVudChodG1sX2IyZTc2NDViNjQ0NTQyOWNhYTJlMjkwMjE2ZTE4YWIyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2NhODJmOTYyNjc1MjRmOGVhZDgzMmJjNDgyMTM1Njc0LmJpbmRQb3B1cChwb3B1cF83MzM2NmY4ZjJhZjc0NDI3YjkzNjJkOTE3NDFhZDc2Zik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8xMTNlN2U0ODAzMDY0MTliOWFiODBlYWMzZmU4ZjYyNiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjk2ODYxMDE5ODYwMzc3LDEwMy44NTA2NjcwMzg0NDAxNF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNThlNDdhMTlkZDhiNDZiZjljMjMxMTViODY1ZDdmNmQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNjYxNDMwNmRiMzZhNGQwMWE4NGExYTkzZGFiNzY2NDAgPSAkKCc8ZGl2IGlkPSJodG1sXzY2MTQzMDZkYjM2YTRkMDFhODRhMWE5M2RhYjc2NjQwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5CcmFzIEJhc2FoIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF81OGU0N2ExOWRkOGI0NmJmOWMyMzExNWI4NjVkN2Y2ZC5zZXRDb250ZW50KGh0bWxfNjYxNDMwNmRiMzZhNGQwMWE4NGExYTkzZGFiNzY2NDApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMTEzZTdlNDgwMzA2NDE5YjlhYjgwZWFjM2ZlOGY2MjYuYmluZFBvcHVwKHBvcHVwXzU4ZTQ3YTE5ZGQ4YjQ2YmY5YzIzMTE1Yjg2NWQ3ZjZkKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzZiNDczOGI3OTVhNTRiNjZhZjMyOGFkNmYyZTQ3NjIzID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMjYzNDQ3MDQ5OTQ0NzU3LDEwMy44OTAyODY2OTgwMDI2N10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNWFkOTE0NmQxNzAzNDFiYzg4YjJhOThjMjQ1NTFiNGIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfY2QxMWI1NTUxY2IyNDRiMTk0NGY4YWE4ZTI5NzA0OGEgPSAkKCc8ZGl2IGlkPSJodG1sX2NkMTFiNTU1MWNiMjQ0YjE5NDRmOGFhOGUyOTcwNDhhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5NYWNwaGVyc29uIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF81YWQ5MTQ2ZDE3MDM0MWJjODhiMmE5OGMyNDU1MWI0Yi5zZXRDb250ZW50KGh0bWxfY2QxMWI1NTUxY2IyNDRiMTk0NGY4YWE4ZTI5NzA0OGEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNmI0NzM4Yjc5NWE1NGI2NmFmMzI4YWQ2ZjJlNDc2MjMuYmluZFBvcHVwKHBvcHVwXzVhZDkxNDZkMTcwMzQxYmM4OGIyYTk4YzI0NTUxYjRiKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2ExMTM0ZThjZTAyMzQ3NjA4ZDM1ZTI1YmIwMTYzNjE4ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yOTI5MzU1NzYxNzg4NDQsMTAzLjg1MjU4NTU1OTAwNTU2XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF82YzMwYmUwNDExOTU0MjMyYjY4NGE0YjA5ZDJlOGUxOCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8yNTliMzcwZjA2N2E0NTg3YjIyMGJiNDk0YjM1NDRhYyA9ICQoJzxkaXYgaWQ9Imh0bWxfMjU5YjM3MGYwNjdhNDU4N2IyMjBiYjQ5NGIzNTQ0YWMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkNpdHkgSGFsbCBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNmMzMGJlMDQxMTk1NDIzMmI2ODRhNGIwOWQyZThlMTguc2V0Q29udGVudChodG1sXzI1OWIzNzBmMDY3YTQ1ODdiMjIwYmI0OTRiMzU0NGFjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2ExMTM0ZThjZTAyMzQ3NjA4ZDM1ZTI1YmIwMTYzNjE4LmJpbmRQb3B1cChwb3B1cF82YzMwYmUwNDExOTU0MjMyYjY4NGE0YjA5ZDJlOGUxOCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl84OTY1ODc4ZTBkNmQ0MjQ5OThmMmI1OGJjZjEzNjY1YyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzU2MTkwODE2MzcxNTUwNiwxMDMuOTU0NjM0MTI5NDA5OTJdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzRhMDRlNDE4NjVmZTQxMTFiZGRmODNlMGQzMzU1MGY2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzAyZjg2ODU0MWNiYzQ0YWVhY2U5ZDY1ZWUwMzEwMzYyID0gJCgnPGRpdiBpZD0iaHRtbF8wMmY4Njg1NDFjYmM0NGFlYWNlOWQ2NWVlMDMxMDM2MiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+VGFtcGluZXMgRWFzdCBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNGEwNGU0MTg2NWZlNDExMWJkZGY4M2UwZDMzNTUwZjYuc2V0Q29udGVudChodG1sXzAyZjg2ODU0MWNiYzQ0YWVhY2U5ZDY1ZWUwMzEwMzYyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzg5NjU4NzhlMGQ2ZDQyNDk5OGYyYjU4YmNmMTM2NjVjLmJpbmRQb3B1cChwb3B1cF80YTA0ZTQxODY1ZmU0MTExYmRkZjgzZTBkMzM1NTBmNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl81OTBmOTU4M2JjZjU0YTdkYTM2MzlhNTViODE5YjE5MyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzQ1NTE0NjM4NjM1ODA3LDEwMy45Mzg0MzY2Mzc4OTM2M10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNTA2NTcxZTNkZDA5NGEzMDgyNTM2YTYxNGJjN2ExNmIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYjYwMjkwY2E0NDQ5NDYxNTg1ZjA2ZjllM2M2NWFiMjMgPSAkKCc8ZGl2IGlkPSJodG1sX2I2MDI5MGNhNDQ0OTQ2MTU4NWYwNmY5ZTNjNjVhYjIzIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5UYW1waW5lcyBXZXN0IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF81MDY1NzFlM2RkMDk0YTMwODI1MzZhNjE0YmM3YTE2Yi5zZXRDb250ZW50KGh0bWxfYjYwMjkwY2E0NDQ5NDYxNTg1ZjA2ZjllM2M2NWFiMjMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNTkwZjk1ODNiY2Y1NGE3ZGEzNjM5YTU1YjgxOWIxOTMuYmluZFBvcHVwKHBvcHVwXzUwNjU3MWUzZGQwOTRhMzA4MjUzNmE2MTRiYzdhMTZiKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2IxOGYzMmJmNmMyZjRlMzVhMjUwZTM1Mzk4MDFmNjg3ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMjE1MDUxNzEzNTA2MTcyLDEwMy44NzE5MDAyOTIzNTg0Ml0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYTcxODk4OWJkNTgzNDM3OWFmZmIwM2MzYjAzMzliMDUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYWEwZWE5M2UxZTdjNDQzMTljZjkzNDk4MDc0ZDBkNmMgPSAkKCc8ZGl2IGlkPSJodG1sX2FhMGVhOTNlMWU3YzQ0MzE5Y2Y5MzQ5ODA3NGQwZDZjIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5HZXlsYW5nIEJhaHJ1IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9hNzE4OTg5YmQ1ODM0Mzc5YWZmYjAzYzNiMDMzOWIwNS5zZXRDb250ZW50KGh0bWxfYWEwZWE5M2UxZTdjNDQzMTljZjkzNDk4MDc0ZDBkNmMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfYjE4ZjMyYmY2YzJmNGUzNWEyNTBlMzUzOTgwMWY2ODcuYmluZFBvcHVwKHBvcHVwX2E3MTg5ODliZDU4MzQzNzlhZmZiMDNjM2IwMzM5YjA1KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2UxNGM1YmIwZjY4YTQyMjlhYmQzZGE4MjBiZGE5ZjU1ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMTM2NzE1NjU4Nzc0MDU5LDEwMy44NjI5Nzc2NTQxMzE0M10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOTQ0Y2Q1MmViZDQzNGUwMzhmNWQ4ZTgxMGRjZWFmNzEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfN2VkYjQ1NTdmYmNjNDM1YzkyOTQyYWJkYTEwMmEzOTcgPSAkKCc8ZGl2IGlkPSJodG1sXzdlZGI0NTU3ZmJjYzQzNWM5Mjk0MmFiZGExMDJhMzk3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5CZW5kZW1lZXIgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzk0NGNkNTJlYmQ0MzRlMDM4ZjVkOGU4MTBkY2VhZjcxLnNldENvbnRlbnQoaHRtbF83ZWRiNDU1N2ZiY2M0MzVjOTI5NDJhYmRhMTAyYTM5Nyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9lMTRjNWJiMGY2OGE0MjI5YWJkM2RhODIwYmRhOWY1NS5iaW5kUG9wdXAocG9wdXBfOTQ0Y2Q1MmViZDQzNGUwMzhmNWQ4ZTgxMGRjZWFmNzEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfYjNhMjcyZTExMjdjNGQ0YmI1NzkzMjA2YzM5OTViMWYgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjI5ODg2MzYwMzc1MDEzMjcsMTAzLjg1MDM4MzkyMTU3ODU3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF83ODZjYTZmYjg0Yzk0Zjc2ODA4NWEwYTNkMDgwOGVmOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9iMzEwYzQ1YTI1ZTc0MjRiOGZkYzNkNWEwZWEyNzcwMCA9ICQoJzxkaXYgaWQ9Imh0bWxfYjMxMGM0NWEyNWU3NDI0YjhmZGMzZDVhMGVhMjc3MDAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkJlbmNvb2xlbiBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNzg2Y2E2ZmI4NGM5NGY3NjgwODVhMGEzZDA4MDhlZjkuc2V0Q29udGVudChodG1sX2IzMTBjNDVhMjVlNzQyNGI4ZmRjM2Q1YTBlYTI3NzAwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2IzYTI3MmUxMTI3YzRkNGJiNTc5MzIwNmMzOTk1YjFmLmJpbmRQb3B1cChwb3B1cF83ODZjYTZmYjg0Yzk0Zjc2ODA4NWEwYTNkMDgwOGVmOSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9lN2RjMTZmMmQ1NDg0YzI5YmJkMGEwZDFhNmRkMTYyMCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjY1NDcxOTczMTk0Njg0OSwxMDMuODIxNDQyNzM1NzU4Ml0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYmZjMjQ4YjE0ZGY1NGZjY2IxZTM2YzNiZDQ3OTFjZDIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNjdhOTczMTNjMjA2NDYwYThmYTYzMDBiOTQ2ZjBlZTIgPSAkKCc8ZGl2IGlkPSJodG1sXzY3YTk3MzEzYzIwNjQ2MGE4ZmE2MzAwYjk0NmYwZWUyIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5IYXJib3VyZnJvbnQgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2JmYzI0OGIxNGRmNTRmY2NiMWUzNmMzYmQ0NzkxY2QyLnNldENvbnRlbnQoaHRtbF82N2E5NzMxM2MyMDY0NjBhOGZhNjMwMGI5NDZmMGVlMik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9lN2RjMTZmMmQ1NDg0YzI5YmJkMGEwZDFhNmRkMTYyMC5iaW5kUG9wdXAocG9wdXBfYmZjMjQ4YjE0ZGY1NGZjY2IxZTM2YzNiZDQ3OTFjZDIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMWNmYzAzN2ZlZWU1NGRmNGI4ZGIxNGJlYjA5MDc2MmIgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjI5Mjg5MDg4NTQyNDk2MzksMTAzLjg2MDg1MjY1NDkxNDldLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzA5MTAwYjJiZDVkNjRkOTc5NjA0N2ZhMjE2MDUxZWE0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzQ0MzVjMzY3MThhMDRjYjNhOTZiNjNkYTBlM2EzNmQ5ID0gJCgnPGRpdiBpZD0iaHRtbF80NDM1YzM2NzE4YTA0Y2IzYTk2YjYzZGEwZTNhMzZkOSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+UHJvbWVuYWRlIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wOTEwMGIyYmQ1ZDY0ZDk3OTYwNDdmYTIxNjA1MWVhNC5zZXRDb250ZW50KGh0bWxfNDQzNWMzNjcxOGEwNGNiM2E5NmI2M2RhMGUzYTM2ZDkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMWNmYzAzN2ZlZWU1NGRmNGI4ZGIxNGJlYjA5MDc2MmIuYmluZFBvcHVwKHBvcHVwXzA5MTAwYjJiZDVkNjRkOTc5NjA0N2ZhMjE2MDUxZWE0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2UzMjQzMDA2NzU5MDQ2NDY4YzNhODY4NmFjZTk3ZGNjID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yOTk1NTAwNzkwNDIxOTc5LDEwMy44NTY4Njc5OTk3NTU3Ml0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNzk0YjliNjBmMzQ0NGY0NDg5MGFhZjcwZjM0MTk1YjcgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfODI4MWM0Y2ZlNjJmNGI1YmEzODYwMTE2ZWUzODZkODEgPSAkKCc8ZGl2IGlkPSJodG1sXzgyODFjNGNmZTYyZjRiNWJhMzg2MDExNmVlMzg2ZDgxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5CdWdpcyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNzk0YjliNjBmMzQ0NGY0NDg5MGFhZjcwZjM0MTk1Yjcuc2V0Q29udGVudChodG1sXzgyODFjNGNmZTYyZjRiNWJhMzg2MDExNmVlMzg2ZDgxKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2UzMjQzMDA2NzU5MDQ2NDY4YzNhODY4NmFjZTk3ZGNjLmJpbmRQb3B1cChwb3B1cF83OTRiOWI2MGYzNDQ0ZjQ0ODkwYWFmNzBmMzQxOTViNyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl84ZTJiNmY4YjM4ZTg0NWEzODFmMmVlZDhjODg3NTJiMCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjkzNDYxOTY2NDc3NDY0LDEwMy43ODQ2MzEwOTM0ODEzNV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYWQ1ZjZmODUyYTBlNDZmYmIyZTAyNGY5NjVjYzFmMGYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYzEzZTM0ZmIwZGFkNGJiNTlkNDllOTg5NjdkMTJhNmMgPSAkKCc8ZGl2IGlkPSJodG1sX2MxM2UzNGZiMGRhZDRiYjU5ZDQ5ZTk4OTY3ZDEyYTZjIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5LZW50IFJpZGdlIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9hZDVmNmY4NTJhMGU0NmZiYjJlMDI0Zjk2NWNjMWYwZi5zZXRDb250ZW50KGh0bWxfYzEzZTM0ZmIwZGFkNGJiNTlkNDllOTg5NjdkMTJhNmMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfOGUyYjZmOGIzOGU4NDVhMzgxZjJlZWQ4Yzg4NzUyYjAuYmluZFBvcHVwKHBvcHVwX2FkNWY2Zjg1MmEwZTQ2ZmJiMmUwMjRmOTY1Y2MxZjBmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzEyYzhmZjYwMWVmZDRhYTE4YjBmOTAzYTY2N2EwOTk2ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yNzY1MjA1ODAwMjU2ODYyLDEwMy44NDU4NjM4OTYwMTA4M10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfM2QwM2U0Yzg4Y2IwNGExNGI5NmY5MDAzYmQ0MzQ3MWMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNTg1ZTU5YzY0MmZjNGZjNzhiNjIyMmQ0ZGE0ZjQzMjggPSAkKCc8ZGl2IGlkPSJodG1sXzU4NWU1OWM2NDJmYzRmYzc4YjYyMjJkNGRhNGY0MzI4IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5UYW5qb25nIFBhZ2FyIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8zZDAzZTRjODhjYjA0YTE0Yjk2ZjkwMDNiZDQzNDcxYy5zZXRDb250ZW50KGh0bWxfNTg1ZTU5YzY0MmZjNGZjNzhiNjIyMmQ0ZGE0ZjQzMjgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMTJjOGZmNjAxZWZkNGFhMThiMGY5MDNhNjY3YTA5OTYuYmluZFBvcHVwKHBvcHVwXzNkMDNlNGM4OGNiMDRhMTRiOTZmOTAwM2JkNDM0NzFjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzk0MDUzZmMyOTFkYzQ1ZWViMDM0MjM3N2RiNGI3NDBhID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yOTk3NTkyMTE5MjQ5NDksMTAzLjc4NzQ1NzE3NDUxNzY4XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xYjhmNzdjZWM2YjI0YzgzOWUyZDI2MGQ5MmVkODJlMyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85ZDcwZmM0MjFkNTg0MzA3YTQ1YWM0OTM5MzRlZmY1NSA9ICQoJzxkaXYgaWQ9Imh0bWxfOWQ3MGZjNDIxZDU4NDMwN2E0NWFjNDkzOTM0ZWZmNTUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPk9uZS1Ob3J0aCBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMWI4Zjc3Y2VjNmIyNGM4MzllMmQyNjBkOTJlZDgyZTMuc2V0Q29udGVudChodG1sXzlkNzBmYzQyMWQ1ODQzMDdhNDVhYzQ5MzkzNGVmZjU1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzk0MDUzZmMyOTFkYzQ1ZWViMDM0MjM3N2RiNGI3NDBhLmJpbmRQb3B1cChwb3B1cF8xYjhmNzdjZWM2YjI0YzgzOWUyZDI2MGQ5MmVkODJlMyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8xMmY0MjJlOTgzNDE0ZWE5YjY1MWJlZTBlZGIzNDFjMCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzUxMzA4MDEzNDQ4OTQ0NCwxMDMuODQ5MTU0MDg1Njg0MzZdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2ZmY2QwOTMzNDE5NDQwZTFhMDk1YWMxZmViOGI2MGY1ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzg2ODg2ZTFlZGM4NzQxYjViOGM2NWQyNGYxN2VhZmZiID0gJCgnPGRpdiBpZD0iaHRtbF84Njg4NmUxZWRjODc0MWI1YjhjNjVkMjRmMTdlYWZmYiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+QmlzaGFuIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9mZmNkMDkzMzQxOTQ0MGUxYTA5NWFjMWZlYjhiNjBmNS5zZXRDb250ZW50KGh0bWxfODY4ODZlMWVkYzg3NDFiNWI4YzY1ZDI0ZjE3ZWFmZmIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMTJmNDIyZTk4MzQxNGVhOWI2NTFiZWUwZWRiMzQxYzAuYmluZFBvcHVwKHBvcHVwX2ZmY2QwOTMzNDE5NDQwZTFhMDk1YWMxZmViOGI2MGY1KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzI4NmQ0N2RjZGE4MjRhNTE5YTUyZjY0ZjQ3MDNlMjYzID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMDI4Mzk5NjI5NjYxNjg5LDEwMy44NzUzNTAwMDE4Mjg2Nl0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZGQ2MmY1ODE3NTBmNDM3NWFhYzViZTEwNDJhMWE2NTkgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYjFkNTYwMDFmNGViNGQyZWI0ZjVmNTE2N2Y3NWE0ODEgPSAkKCc8ZGl2IGlkPSJodG1sX2IxZDU2MDAxZjRlYjRkMmViNGY1ZjUxNjdmNzVhNDgxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TdGFkaXVtIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9kZDYyZjU4MTc1MGY0Mzc1YWFjNWJlMTA0MmExYTY1OS5zZXRDb250ZW50KGh0bWxfYjFkNTYwMDFmNGViNGQyZWI0ZjVmNTE2N2Y3NWE0ODEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMjg2ZDQ3ZGNkYTgyNGE1MTlhNTJmNjRmNDcwM2UyNjMuYmluZFBvcHVwKHBvcHVwX2RkNjJmNTgxNzUwZjQzNzVhYWM1YmUxMDQyYTFhNjU5KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2VjYWQ1ZTQ1Yjg3MzRkYTU4OTJlYjgyZTZjYjEwYzdjID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMDAyNTkzODgzOTgxOTI2LDEwMy44MzkwNzQ4MjkwMDkzXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8wNWEyN2U3YmUyNzI0MjkyYjliZjVkZGI5MjNlNDc1ZCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85YTgyY2M2OTIzM2Y0YzJjYTE1ZmVjNzRhMTFlZGE0NCA9ICQoJzxkaXYgaWQ9Imh0bWxfOWE4MmNjNjkyMzNmNGMyY2ExNWZlYzc0YTExZWRhNDQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNvbWVyc2V0IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wNWEyN2U3YmUyNzI0MjkyYjliZjVkZGI5MjNlNDc1ZC5zZXRDb250ZW50KGh0bWxfOWE4MmNjNjkyMzNmNGMyY2ExNWZlYzc0YTExZWRhNDQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZWNhZDVlNDViODczNGRhNTg5MmViODJlNmNiMTBjN2MuYmluZFBvcHVwKHBvcHVwXzA1YTI3ZTdiZTI3MjQyOTJiOWJmNWRkYjkyM2U0NzVkKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzdlMWQwMDIyMTA0ODQ5NTdiODY1ZTRhZmY2NmYwMjA4ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMDM5ODAzNDU3Njk2NDczLDEwMy44MzIyNDExMTg3MzQ3M10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYjA1YzhjZGRjNDQxNGU0ODgwYTA1ZDgzOGZjNGM3Y2IgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYmE5NmQ1N2VjM2U3NGYyYmJjMzhmZmUwZDVmZDg0NGYgPSAkKCc8ZGl2IGlkPSJodG1sX2JhOTZkNTdlYzNlNzRmMmJiYzM4ZmZlMGQ1ZmQ4NDRmIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5PcmNoYXJkIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iMDVjOGNkZGM0NDE0ZTQ4ODBhMDVkODM4ZmM0YzdjYi5zZXRDb250ZW50KGh0bWxfYmE5NmQ1N2VjM2U3NGYyYmJjMzhmZmUwZDVmZDg0NGYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfN2UxZDAwMjIxMDQ4NDk1N2I4NjVlNGFmZjY2ZjAyMDguYmluZFBvcHVwKHBvcHVwX2IwNWM4Y2RkYzQ0MTRlNDg4MGEwNWQ4MzhmYzRjN2NiKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2Y0MjE4MTlmMzE2NjRlNGRiNDc3YWRkNTk1Zjg3MzhhID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMDU0MDI5NzU1NTYwOCwxMDMuODU1NDc2NjgyOTk5NTJdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzI5MjU1MzhmNjFiMjRjZGQ4NmViOGY1ZGY0YTQwMGMyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzZlNjdjYmY2MzAxZDQ3YmZiOGRjZTc4Mzk0NmJiYmEzID0gJCgnPGRpdiBpZD0iaHRtbF82ZTY3Y2JmNjMwMWQ0N2JmYjhkY2U3ODM5NDZiYmJhMyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+SmFsYW4gQmVzYXIgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzI5MjU1MzhmNjFiMjRjZGQ4NmViOGY1ZGY0YTQwMGMyLnNldENvbnRlbnQoaHRtbF82ZTY3Y2JmNjMwMWQ0N2JmYjhkY2U3ODM5NDZiYmJhMyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9mNDIxODE5ZjMxNjY0ZTRkYjQ3N2FkZDU5NWY4NzM4YS5iaW5kUG9wdXAocG9wdXBfMjkyNTUzOGY2MWIyNGNkZDg2ZWI4ZjVkZjRhNDAwYzIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMjQ1ZjRiZjIzYjQyNDE3NjhkMjA0NTlkYjg2OTMzMmQgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjI4MTg3MzEyMTgyNDI2MjMsMTAzLjg1OTA3OTQzMTc5MTMzXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8zODNkMWIyN2EzYzk0NDFjOTE4NjkzZmQ0ODZmZDczNSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9kNjczOTUxOWYwNDc0YzM0OGFjNmYwYTcwMWFiMWE5NyA9ICQoJzxkaXYgaWQ9Imh0bWxfZDY3Mzk1MTlmMDQ3NGMzNDhhYzZmMGE3MDFhYjFhOTciIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkJheWZyb250IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8zODNkMWIyN2EzYzk0NDFjOTE4NjkzZmQ0ODZmZDczNS5zZXRDb250ZW50KGh0bWxfZDY3Mzk1MTlmMDQ3NGMzNDhhYzZmMGE3MDFhYjFhOTcpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMjQ1ZjRiZjIzYjQyNDE3NjhkMjA0NTlkYjg2OTMzMmQuYmluZFBvcHVwKHBvcHVwXzM4M2QxYjI3YTNjOTQ0MWM5MTg2OTNmZDQ4NmZkNzM1KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzdmMGUxNDlmM2RjMzQzODViZTk2NjQzYjk3MTRmNGI2ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMzM3MjgyMTU2NTQzNjgyLDEwMy44MzA2ODkyMTY4MDQyM10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDQ1ZjM4MTBkMWU1NDE3ZTgzYjE3MzNjNDQzNmNmOWYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMzFlZTg4MWVlNThiNDllNWEwMDhlOTYxYWE0NWNlZjAgPSAkKCc8ZGl2IGlkPSJodG1sXzMxZWU4ODFlZTU4YjQ5ZTVhMDA4ZTk2MWFhNDVjZWYwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5CdWtpdCBCcm93biBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNDQ1ZjM4MTBkMWU1NDE3ZTgzYjE3MzNjNDQzNmNmOWYuc2V0Q29udGVudChodG1sXzMxZWU4ODFlZTU4YjQ5ZTVhMDA4ZTk2MWFhNDVjZWYwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzdmMGUxNDlmM2RjMzQzODViZTk2NjQzYjk3MTRmNGI2LmJpbmRQb3B1cChwb3B1cF80NDVmMzgxMGQxZTU0MTdlODNiMTczM2M0NDM2Y2Y5Zik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9jZjBmYjI4NDg5YWY0ZDYxOWFmOTRkZWJmYmNmOTNkMyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzA2MjAxMjM4MTY0MjY1NCwxMDMuODgyNTI3NzQ3NDU0MzJdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2NlOGRiOWU3MDkzYTQ1MzhiMGYzMWMxN2I4NTkzOWU0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzRiMDQ5MWI0YzUwMTQyOGQ5NGMzZDA1YjU5MjYzZGQ3ID0gJCgnPGRpdiBpZD0iaHRtbF80YjA0OTFiNGM1MDE0MjhkOTRjM2QwNWI1OTI2M2RkNyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+TW91bnRiYXR0ZW4gTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2NlOGRiOWU3MDkzYTQ1MzhiMGYzMWMxN2I4NTkzOWU0LnNldENvbnRlbnQoaHRtbF80YjA0OTFiNGM1MDE0MjhkOTRjM2QwNWI1OTI2M2RkNyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9jZjBmYjI4NDg5YWY0ZDYxOWFmOTRkZWJmYmNmOTNkMy5iaW5kUG9wdXAocG9wdXBfY2U4ZGI5ZTcwOTNhNDUzOGIwZjMxYzE3Yjg1OTM5ZTQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMDU2YmY0OWQ1ZTViNDMzY2I4YTkwNDA4NzY1ZGMyMTkgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM2MDE3ODUwNDUxMjQ5MDYsMTAzLjg4NTA2NDUyMjk3NzAyXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yZWY1ZjIzYWJiZmU0YmFhYjRkM2ZhZWYxODY1Y2ZmNiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83NjNmMzkwOGFmZmI0ZWYyYTNkNDY2YjBhNjQyYzM1MiA9ICQoJzxkaXYgaWQ9Imh0bWxfNzYzZjM5MDhhZmZiNGVmMmEzZDQ2NmIwYTY0MmMzNTIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPktvdmFuIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yZWY1ZjIzYWJiZmU0YmFhYjRkM2ZhZWYxODY1Y2ZmNi5zZXRDb250ZW50KGh0bWxfNzYzZjM5MDhhZmZiNGVmMmEzZDQ2NmIwYTY0MmMzNTIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMDU2YmY0OWQ1ZTViNDMzY2I4YTkwNDA4NzY1ZGMyMTkuYmluZFBvcHVwKHBvcHVwXzJlZjVmMjNhYmJmZTRiYWFiNGQzZmFlZjE4NjVjZmY2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2QwOTk0OWI4NGZhMjRlZDNhMDNjZGRjOGIzNjY1MzEzID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yOTI0NzgyNjEyNTAyMDUsMTAzLjg0NDMyNzkzNjI3MzQ3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xNjUxYTc5NWI4NDM0YWQ1OGJhMjBlZjE3YThmMGUwNCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9iNzE2ZGZiMGJjNzE0ZjQwODIzNTZlNjUwOTRhMGRkZCA9ICQoJzxkaXYgaWQ9Imh0bWxfYjcxNmRmYjBiYzcxNGY0MDgyMzU2ZTY1MDk0YTBkZGQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkZvcnQgQ2FubmluZyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMTY1MWE3OTViODQzNGFkNThiYTIwZWYxN2E4ZjBlMDQuc2V0Q29udGVudChodG1sX2I3MTZkZmIwYmM3MTRmNDA4MjM1NmU2NTA5NGEwZGRkKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2QwOTk0OWI4NGZhMjRlZDNhMDNjZGRjOGIzNjY1MzEzLmJpbmRQb3B1cChwb3B1cF8xNjUxYTc5NWI4NDM0YWQ1OGJhMjBlZjE3YThmMGUwNCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9iNzkyMTI5YzU1YzM0YjNlYWM4OWUyZWMxYmI5NWJjYSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzE3NTA5OTQ1MjI4NjczLDEwMy44MDc1ODU3NzI0MjE0OF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOThiYzYxYjY5NDdiNDFhNmI0ZTc0MTZjZTM5MzhlMjggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZGY5NmQ0NTA3MTAzNDNkNzlmYTllZWY2MzMwY2Y2MjUgPSAkKCc8ZGl2IGlkPSJodG1sX2RmOTZkNDUwNzEwMzQzZDc5ZmE5ZWVmNjMzMGNmNjI1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5GYXJyZXIgUm9hZCBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOThiYzYxYjY5NDdiNDFhNmI0ZTc0MTZjZTM5MzhlMjguc2V0Q29udGVudChodG1sX2RmOTZkNDUwNzEwMzQzZDc5ZmE5ZWVmNjMzMGNmNjI1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2I3OTIxMjljNTVjMzRiM2VhYzg5ZTJlYzFiYjk1YmNhLmJpbmRQb3B1cChwb3B1cF85OGJjNjFiNjk0N2I0MWE2YjRlNzQxNmNlMzkzOGUyOCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8xMDIxNjc1M2QzZGY0YmFlODY0ZDlmZTYyZmM4OGE3YSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjc5NDQ1NTIzNzExNDA1MywxMDMuODUyODQwMTEwNzAwNzVdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzIwNWI0ODlhN2VjMzQyMDQ5NWM1ZjcyYmEyZDZjZTQ1ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzhlOTZiNTVkZjExOTQ5ODViMzEzZTNkNzQ5ODU3NzI5ID0gJCgnPGRpdiBpZD0iaHRtbF84ZTk2YjU1ZGYxMTk0OTg1YjMxM2UzZDc0OTg1NzcyOSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+RG93bnRvd24gTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzIwNWI0ODlhN2VjMzQyMDQ5NWM1ZjcyYmEyZDZjZTQ1LnNldENvbnRlbnQoaHRtbF84ZTk2YjU1ZGYxMTk0OTg1YjMxM2UzZDc0OTg1NzcyOSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl8xMDIxNjc1M2QzZGY0YmFlODY0ZDlmZTYyZmM4OGE3YS5iaW5kUG9wdXAocG9wdXBfMjA1YjQ4OWE3ZWMzNDIwNDk1YzVmNzJiYTJkNmNlNDUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZTY5OTdhMjQ1OTVjNDNhZGJlZjIzNmZhZWM5ZWY2YTUgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjI5MzMyMDk0MDk0NDAwMywxMDMuODU1NTAzNzcyMTg2OTVdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzJiMzBiN2UwYzBkMTQ4MDg4OWY4NzE0NjUyMzBmMTkzID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2U2MzgwMmNlMTEwYjRlODliMjc3MTU5YmUyYjc2MjQ4ID0gJCgnPGRpdiBpZD0iaHRtbF9lNjM4MDJjZTExMGI0ZTg5YjI3NzE1OWJlMmI3NjI0OCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+RXNwbGFuYWRlIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yYjMwYjdlMGMwZDE0ODA4ODlmODcxNDY1MjMwZjE5My5zZXRDb250ZW50KGh0bWxfZTYzODAyY2UxMTBiNGU4OWIyNzcxNTliZTJiNzYyNDgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZTY5OTdhMjQ1OTVjNDNhZGJlZjIzNmZhZWM5ZWY2YTUuYmluZFBvcHVwKHBvcHVwXzJiMzBiN2UwYzBkMTQ4MDg4OWY4NzE0NjUyMzBmMTkzKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzMyMWY0NThiNzMyZDQyMWZhOGFmNjBhMDRjN2NlMDFiID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yOTk3NjYxNjg2MDA4MjAyLDEwMy44NjM2MzY2ODUxNTI4OF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZTAwZWM4N2ZmYWQ2NDNjNDg5NGI4MjczZTY4ODAwNzQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMjI0OWFhYTVjMDY2NDQwMDhjZTRjMDg3MmZhYzk0NDkgPSAkKCc8ZGl2IGlkPSJodG1sXzIyNDlhYWE1YzA2NjQ0MDA4Y2U0YzA4NzJmYWM5NDQ5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5OaWNvbGwgSGlnaHdheSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZTAwZWM4N2ZmYWQ2NDNjNDg5NGI4MjczZTY4ODAwNzQuc2V0Q29udGVudChodG1sXzIyNDlhYWE1YzA2NjQ0MDA4Y2U0YzA4NzJmYWM5NDQ5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzMyMWY0NThiNzMyZDQyMWZhOGFmNjBhMDRjN2NlMDFiLmJpbmRQb3B1cChwb3B1cF9lMDBlYzg3ZmZhZDY0M2M0ODk0YjgyNzNlNjg4MDA3NCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9jOTBlMWIzYjViNDY0NjMzOWNkYTZjMmM2MDZiMDY5MiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzM3NjczODQxMzc3Mjg4NiwxMDMuODM5NTI5NDg3NDI5MDldLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzQyZTQ3NWI0YmYyNDRmYTE4NDg3Y2IwYWFmZGUxZThkID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzRkYTYzNWI1OWIyNTRhMmJhZTE5ZWI5NTk3NzQwMDIxID0gJCgnPGRpdiBpZD0iaHRtbF80ZGE2MzViNTliMjU0YTJiYWUxOWViOTU5Nzc0MDAyMSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+Q2FsZGVjb3R0IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80MmU0NzViNGJmMjQ0ZmExODQ4N2NiMGFhZmRlMWU4ZC5zZXRDb250ZW50KGh0bWxfNGRhNjM1YjU5YjI1NGEyYmFlMTllYjk1OTc3NDAwMjEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfYzkwZTFiM2I1YjQ2NDYzMzljZGE2YzJjNjA2YjA2OTIuYmluZFBvcHVwKHBvcHVwXzQyZTQ3NWI0YmYyNDRmYTE4NDg3Y2IwYWFmZGUxZThkKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2U1NDczYTMxOGQ1NTQ0NTFhNmQ2MTdhOTBhZDM3ZTg5ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yNzY0MjY2ODgwODMyNzgsMTAzLjg1NDU5NzQ0NDIwNDAzXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF80OGJiZmM4ZDYyYmY0NDdkYmQ2NTc4OTQyMjYxNDk5ZiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80MDI1MmZlNWQyZTQ0YWY2OWZkOWEzYWQzMzRlNTQ2OCA9ICQoJzxkaXYgaWQ9Imh0bWxfNDAyNTJmZTVkMmU0NGFmNjlmZDlhM2FkMzM0ZTU0NjgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPk1hcmluYSBCYXkgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzQ4YmJmYzhkNjJiZjQ0N2RiZDY1Nzg5NDIyNjE0OTlmLnNldENvbnRlbnQoaHRtbF80MDI1MmZlNWQyZTQ0YWY2OWZkOWEzYWQzMzRlNTQ2OCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9lNTQ3M2EzMThkNTU0NDUxYTZkNjE3YTkwYWQzN2U4OS5iaW5kUG9wdXAocG9wdXBfNDhiYmZjOGQ2MmJmNDQ3ZGJkNjU3ODk0MjI2MTQ5OWYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZjdjZjBhNmY4ZDQ1NDhkY2JhOTNkNDU4OWVlOWQxMjMgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjI3MjMzMjA2NDkyOTE3NywxMDMuODAyOTM5Mjg3MDk2NTVdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzIyNDYxZGJkNDA0MzQwZmE4YzA0YTA3ODA5OTVjYmJhID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzhiZWQ1Y2FmZWRiMDRjYmZiYTdlMWU0OTdkYjJkZWFhID0gJCgnPGRpdiBpZD0iaHRtbF84YmVkNWNhZmVkYjA0Y2JmYmE3ZTFlNDk3ZGIyZGVhYSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+TGFicmFkb3IgUGFyayBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMjI0NjFkYmQ0MDQzNDBmYThjMDRhMDc4MDk5NWNiYmEuc2V0Q29udGVudChodG1sXzhiZWQ1Y2FmZWRiMDRjYmZiYTdlMWU0OTdkYjJkZWFhKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2Y3Y2YwYTZmOGQ0NTQ4ZGNiYTkzZDQ1ODllZTlkMTIzLmJpbmRQb3B1cChwb3B1cF8yMjQ2MWRiZDQwNDM0MGZhOGMwNGEwNzgwOTk1Y2JiYSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8yY2Q0YzI1M2JlNjk0ZTFhODUwZjU0Y2JkOGRkY2NhZiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjg0MzU4OTExNTY1NjU4OCwxMDMuODQzNDI2NDQyNjA4MDVdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzdlYjlmOTg5YmU2MzRlNDdiNjcxZWYyYjBiYzdkN2UyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2I2OGUxYTM3MTQ4MDRkZmRiN2ZkN2UyYTI4MDY5YzhkID0gJCgnPGRpdiBpZD0iaHRtbF9iNjhlMWEzNzE0ODA0ZGZkYjdmZDdlMmEyODA2OWM4ZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+Q2hpbmF0b3duIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF83ZWI5Zjk4OWJlNjM0ZTQ3YjY3MWVmMmIwYmM3ZDdlMi5zZXRDb250ZW50KGh0bWxfYjY4ZTFhMzcxNDgwNGRmZGI3ZmQ3ZTJhMjgwNjljOGQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMmNkNGMyNTNiZTY5NGUxYTg1MGY1NGNiZDhkZGNjYWYuYmluZFBvcHVwKHBvcHVwXzdlYjlmOTg5YmU2MzRlNDdiNjcxZWYyYjBiYzdkN2UyKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2M1YzcwNGVjMWIyYTRlYTY4YzRmZmU4ZGM4MDcyMDFkID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMTM2MDY0MzU1MjY1MDksMTAzLjgzNzgxMDY2MDc4OTkzXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yN2Y4NTA1OTEzMjQ0MmVmYmY5NGJmNzU5MzRiMGFiNiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83YzgyY2NjZDlmZDI0NzRhYmYyNTkwOWU3ZDg5MDljYSA9ICQoJzxkaXYgaWQ9Imh0bWxfN2M4MmNjY2Q5ZmQyNDc0YWJmMjU5MDllN2Q4OTA5Y2EiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPk5ld3RvbiBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMjdmODUwNTkxMzI0NDJlZmJmOTRiZjc1OTM0YjBhYjYuc2V0Q29udGVudChodG1sXzdjODJjY2NkOWZkMjQ3NGFiZjI1OTA5ZTdkODkwOWNhKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2M1YzcwNGVjMWIyYTRlYTY4YzRmZmU4ZGM4MDcyMDFkLmJpbmRQb3B1cChwb3B1cF8yN2Y4NTA1OTEzMjQ0MmVmYmY5NGJmNzU5MzRiMGFiNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl80MTFkNDQ1MWZiNjQ0NDRiYmRlNGMyYTkyYWZjNDkzNyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzI1ODgyNTQyMjU1Mzk3NSwxMDMuODA3MzIxODc0Mjg3NTRdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzBmNjkzMzEzN2IyOTQzNjdiNmU2ZjhkZWVkNWE4OTYxID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzE4OWIxYTJmNmU4NjRlNjg4NmNjMDU3MDQ0ZGE5ODRmID0gJCgnPGRpdiBpZD0iaHRtbF8xODliMWEyZjZlODY0ZTY4ODZjYzA1NzA0NGRhOTg0ZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+VGFuIEthaCBLZWUgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzBmNjkzMzEzN2IyOTQzNjdiNmU2ZjhkZWVkNWE4OTYxLnNldENvbnRlbnQoaHRtbF8xODliMWEyZjZlODY0ZTY4ODZjYzA1NzA0NGRhOTg0Zik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl80MTFkNDQ1MWZiNjQ0NDRiYmRlNGMyYTkyYWZjNDkzNy5iaW5kUG9wdXAocG9wdXBfMGY2OTMzMTM3YjI5NDM2N2I2ZTZmOGRlZWQ1YTg5NjEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfMmM4ZDIwOWNlZWU2NGVmOGFlNGIwNWQ2MTNmMDRkMjAgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjI4NjE5MjcyNjE5MTY2MzIsMTAzLjgyNzAxOTA5NzU5MjQzXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9mMzNmOGY1ZjlkNGE0ZjliYTJkMjU3NzY1MzkwM2Q4OCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9kYzY0MjNjOGQzNGI0ODJmOTAxMWJlYzM0MmViOGYyOCA9ICQoJzxkaXYgaWQ9Imh0bWxfZGM2NDIzYzhkMzRiNDgyZjkwMTFiZWMzNDJlYjhmMjgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlRpb25nIEJhaHJ1IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9mMzNmOGY1ZjlkNGE0ZjliYTJkMjU3NzY1MzkwM2Q4OC5zZXRDb250ZW50KGh0bWxfZGM2NDIzYzhkMzRiNDgyZjkwMTFiZWMzNDJlYjhmMjgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMmM4ZDIwOWNlZWU2NGVmOGFlNGIwNWQ2MTNmMDRkMjAuYmluZFBvcHVwKHBvcHVwX2YzM2Y4ZjVmOWQ0YTRmOWJhMmQyNTc3NjUzOTAzZDg4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2IzM2I1MDY2N2Q1MjRkYTQ5NjgxOTcxNGIxZTg3ZDVhID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNjIzNDQyMDE4NjEzODI1LDEwMy43Njc0MTc5MjExNjY0XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF82NTg5MzkyZWM1N2M0YTBlYjJkNjkyYzNmMTJhNWNjOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82NDFhOTczNWNmMjc0ZmQ2YmRjMTMzYjc5OGUwMzVkZSA9ICQoJzxkaXYgaWQ9Imh0bWxfNjQxYTk3MzVjZjI3NGZkNmJkYzEzM2I3OThlMDM1ZGUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkhpbGx2aWV3IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF82NTg5MzkyZWM1N2M0YTBlYjJkNjkyYzNmMTJhNWNjOS5zZXRDb250ZW50KGh0bWxfNjQxYTk3MzVjZjI3NGZkNmJkYzEzM2I3OThlMDM1ZGUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfYjMzYjUwNjY3ZDUyNGRhNDk2ODE5NzE0YjFlODdkNWEuYmluZFBvcHVwKHBvcHVwXzY1ODkzOTJlYzU3YzRhMGViMmQ2OTJjM2YxMmE1Y2M5KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzA5NDg0ZTI1YjMxNjRjYmM4MjRkZTg1ZGNjMDg2NGVjID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMzU2NjQ0NTQwOTI5NTE2LDEwMy43ODM4MDYzMjU1NjM5Ml0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMDdlZTA1ZWMxMjgyNDc5ZmI0M2Y3NmZhNzkyOTc4MzYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNjNlODAyNGIyOWEwNDliOWIxOTIzODIyMmM4NDNkMzEgPSAkKCc8ZGl2IGlkPSJodG1sXzYzZTgwMjRiMjlhMDQ5YjliMTkyMzgyMjJjODQzZDMxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5LaW5nIEFsYmVydCBQYXJrIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wN2VlMDVlYzEyODI0NzlmYjQzZjc2ZmE3OTI5NzgzNi5zZXRDb250ZW50KGh0bWxfNjNlODAyNGIyOWEwNDliOWIxOTIzODIyMmM4NDNkMzEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMDk0ODRlMjViMzE2NGNiYzgyNGRlODVkY2MwODY0ZWMuYmluZFBvcHVwKHBvcHVwXzA3ZWUwNWVjMTI4MjQ3OWZiNDNmNzZmYTc5Mjk3ODM2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2Q5NGE5YjMyMDk4YzQxMjM4NjdmNjVjODc5MjViYzBjID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMjY4NzYwNDgwNDU3NTcsMTAzLjg4MzI0NzE3NTgyMzM2XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jMGY1MDYyZDc3MTA0YTU1YTliYzY3NjI5N2ZmNmMyZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9lZDM0ZDJiNzY1ZWI0MDdkYjRjNjg1MGVjZGE3ZjM3NCA9ICQoJzxkaXYgaWQ9Imh0bWxfZWQzNGQyYjc2NWViNDA3ZGI0YzY4NTBlY2RhN2YzNzQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPk1hdHRhciBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYzBmNTA2MmQ3NzEwNGE1NWE5YmM2NzYyOTdmZjZjMmUuc2V0Q29udGVudChodG1sX2VkMzRkMmI3NjVlYjQwN2RiNGM2ODUwZWNkYTdmMzc0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2Q5NGE5YjMyMDk4YzQxMjM4NjdmNjVjODc5MjViYzBjLmJpbmRQb3B1cChwb3B1cF9jMGY1MDYyZDc3MTA0YTU1YTliYzY3NjI5N2ZmNmMyZSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl84OTA2YmU5MDY3MDQ0NWFkOWU5YzE4YjkxNGZlZWExNSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzI5OTU2MTU5MDM3MTkxLDEwMy44OTkyNTIxOTE2MDI4NF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNWQ0MjFkZDI0N2Q1NDU3MzgyZGM0NThhOWVkNzlmNzIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMjU5NjY5N2Q5N2U1NGNhZWFiZDIxY2RhY2Q4MGExOTAgPSAkKCc8ZGl2IGlkPSJodG1sXzI1OTY2OTdkOTdlNTRjYWVhYmQyMWNkYWNkODBhMTkwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5VYmkgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzVkNDIxZGQyNDdkNTQ1NzM4MmRjNDU4YTllZDc5ZjcyLnNldENvbnRlbnQoaHRtbF8yNTk2Njk3ZDk3ZTU0Y2FlYWJkMjFjZGFjZDgwYTE5MCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl84OTA2YmU5MDY3MDQ0NWFkOWU5YzE4YjkxNGZlZWExNS5iaW5kUG9wdXAocG9wdXBfNWQ0MjFkZDI0N2Q1NDU3MzgyZGM0NThhOWVkNzlmNzIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZGVlMjM3NDVmNmVkNGUxN2E1MjE2YWE0NzZlYmNmNTcgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM0MTIyMjUwOTA0NDY1MywxMDMuNzc1NzkzOTUxOTkzMzldLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2VhZDFjZjA1Mzk5MjRjYzZhMjk5YjcyNzAzOTJjMmVjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2JjZjQ4NWQ4Y2Y3MzRkODI5ZTBjOWQ4ZWFiNjQ2MDI2ID0gJCgnPGRpdiBpZD0iaHRtbF9iY2Y0ODVkOGNmNzM0ZDgyOWUwYzlkOGVhYjY0NjAyNiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+QmVhdXR5IFdvcmxkIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lYWQxY2YwNTM5OTI0Y2M2YTI5OWI3MjcwMzkyYzJlYy5zZXRDb250ZW50KGh0bWxfYmNmNDg1ZDhjZjczNGQ4MjllMGM5ZDhlYWI2NDYwMjYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZGVlMjM3NDVmNmVkNGUxN2E1MjE2YWE0NzZlYmNmNTcuYmluZFBvcHVwKHBvcHVwX2VhZDFjZjA1Mzk5MjRjYzZhMjk5YjcyNzAzOTJjMmVjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2YwNDUwOTA3NTQ4OTRmNjFhMzUzNGQ4MDRmYzQ2M2Q3ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMzQ5NjY2MzQ4OTc3OTUzLDEwMy45MDg0NTk0MDEwNzU5XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF83OTBmNDFiYmJiOWE0ZWE1YWE4YTk2MWIwNDg5NmJhNSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8wMWZhNmIxNTNhY2I0MGRlYWU2ODU0MjcyZjhiMmY5MyA9ICQoJzxkaXYgaWQ9Imh0bWxfMDFmYTZiMTUzYWNiNDBkZWFlNjg1NDI3MmY4YjJmOTMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPktha2kgQnVraXQgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzc5MGY0MWJiYmI5YTRlYTVhYThhOTYxYjA0ODk2YmE1LnNldENvbnRlbnQoaHRtbF8wMWZhNmIxNTNhY2I0MGRlYWU2ODU0MjcyZjhiMmY5Myk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9mMDQ1MDkwNzU0ODk0ZjYxYTM1MzRkODA0ZmM0NjNkNy5iaW5kUG9wdXAocG9wdXBfNzkwZjQxYmJiYjlhNGVhNWFhOGE5NjFiMDQ4OTZiYTUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfOGVhMmNmYWJlNTM5NGEzNjg1ZWE0ZWE2NTU4YjFhYjIgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMzNDc0MTQ1MDA5MjA2MjUsMTAzLjkxNzk3Nzk5NjM2NzE2XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iMTcwZjRlYTQ0ZTI0MjM3YmYzY2ViMTJkZGUzYzVlOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9lZWQzYmViNTM4MGY0Y2NjYjE0NDY2MTZiNzA0ZmNjZiA9ICQoJzxkaXYgaWQ9Imh0bWxfZWVkM2JlYjUzODBmNGNjY2IxNDQ2NjE2YjcwNGZjY2YiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkJlZG9rIE5vcnRoIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iMTcwZjRlYTQ0ZTI0MjM3YmYzY2ViMTJkZGUzYzVlOS5zZXRDb250ZW50KGh0bWxfZWVkM2JlYjUzODBmNGNjY2IxNDQ2NjE2YjcwNGZjY2YpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfOGVhMmNmYWJlNTM5NGEzNjg1ZWE0ZWE2NTU4YjFhYjIuYmluZFBvcHVwKHBvcHVwX2IxNzBmNGVhNDRlMjQyMzdiZjNjZWIxMmRkZTNjNWU5KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzA5Njg0YmQ2MTZiNDQ3YjJiN2YzZDgwMDc3MTQwMTU0ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yOTg3MDA2Mzk5NjIyOTA1LDEwMy44NDYxMTQ4MjEzMzk3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9lYmZjZWE1MGY0OWU0NjFjOWMxOGI1ZmExZWU5MGY0MCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8xOWZmMDI0ZjNkZDA0ZWJhOTY3ZDQ4YmM1ODFiMTgzZiA9ICQoJzxkaXYgaWQ9Imh0bWxfMTlmZjAyNGYzZGQwNGViYTk2N2Q0OGJjNTgxYjE4M2YiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkRob2J5IEdoYXV0IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lYmZjZWE1MGY0OWU0NjFjOWMxOGI1ZmExZWU5MGY0MC5zZXRDb250ZW50KGh0bWxfMTlmZjAyNGYzZGQwNGViYTk2N2Q0OGJjNTgxYjE4M2YpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMDk2ODRiZDYxNmI0NDdiMmI3ZjNkODAwNzcxNDAxNTQuYmluZFBvcHVwKHBvcHVwX2ViZmNlYTUwZjQ5ZTQ2MWM5YzE4YjVmYTFlZTkwZjQwKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzliZDdkNTdlMGQxODRhZDc5ZTAzZTUwN2VjNjg0YWIwID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNTA1OTQ1ODk3NzY1MDI2LDEwMy44NzIzNjc3NTYxNTI0XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9lOTRhYTZiNWY3M2I0MWY5YWRiZmI4Njc0M2EwOTIyMiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9mNGUyNzgyZmU5MDc0ODE4YTAwZDkzZTNkNDg5MjM3NSA9ICQoJzxkaXYgaWQ9Imh0bWxfZjRlMjc4MmZlOTA3NDgxOGEwMGQ5M2UzZDQ4OTIzNzUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNlcmFuZ29vbiBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZTk0YWE2YjVmNzNiNDFmOWFkYmZiODY3NDNhMDkyMjIuc2V0Q29udGVudChodG1sX2Y0ZTI3ODJmZTkwNzQ4MThhMDBkOTNlM2Q0ODkyMzc1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzliZDdkNTdlMGQxODRhZDc5ZTAzZTUwN2VjNjg0YWIwLmJpbmRQb3B1cChwb3B1cF9lOTRhYTZiNWY3M2I0MWY5YWRiZmI4Njc0M2EwOTIyMik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl81ODJmMzNiY2IzNzk0NzZhODIxMDJiZDVlZDAzNDc5NCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzM5MTg5Mzc4ODU4MjQ5NiwxMDMuODcwODE3OTc2MDI3NDRdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2FmNDViMTdmZGNlODQwMDI4YzgwNGY4OTgzZjEzZDIwID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzU2NDUxYWFhNjkxODQwYjRhMmQ4ODg5YzA2Y2NjODE4ID0gJCgnPGRpdiBpZD0iaHRtbF81NjQ1MWFhYTY5MTg0MGI0YTJkODg4OWMwNmNjYzgxOCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+V29vZGxlaWdoIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9hZjQ1YjE3ZmRjZTg0MDAyOGM4MDRmODk4M2YxM2QyMC5zZXRDb250ZW50KGh0bWxfNTY0NTFhYWE2OTE4NDBiNGEyZDg4ODljMDZjY2M4MTgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNTgyZjMzYmNiMzc5NDc2YTgyMTAyYmQ1ZWQwMzQ3OTQuYmluZFBvcHVwKHBvcHVwX2FmNDViMTdmZGNlODQwMDI4YzgwNGY4OTgzZjEzZDIwKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2UxZTg1ZTg0OGJhMTQ1YTlhYWM1NWYzMDFmN2I4OWJjID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMjA0NDAxMjQxODA3NjUzLDEwMy44NDM4MjUyODU0Nzk0NF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDk5ZGJmYTBjNjE4NGM4ZDlmNzVkOTlhNGEwOTljYTEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMDA0M2VkZTZkODFhNGJhMTg5ODFkNTI5YmU4NjcyZjAgPSAkKCc8ZGl2IGlkPSJodG1sXzAwNDNlZGU2ZDgxYTRiYTE4OTgxZDUyOWJlODY3MmYwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5Ob3ZlbmEgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzQ5OWRiZmEwYzYxODRjOGQ5Zjc1ZDk5YTRhMDk5Y2ExLnNldENvbnRlbnQoaHRtbF8wMDQzZWRlNmQ4MWE0YmExODk4MWQ1MjliZTg2NzJmMCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9lMWU4NWU4NDhiYTE0NWE5YWFjNTVmMzAxZjdiODliYy5iaW5kUG9wdXAocG9wdXBfNDk5ZGJmYTBjNjE4NGM4ZDlmNzVkOTlhNGEwOTljYTEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfYmVhMjczMGNkYWE0NGIyYjk1NThlOWE2NDM4NmQ5ZjggPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMzNjYwNzE2MzE0NDQzMjMsMTAzLjkzMjIzNDI5MDE4NDI1XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iZGY0ZWJiODFjOWQ0M2NlYmZiZTE3YjIxODk0NzI0ZiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8xMDc0MjZjZmNmMjc0NjY5Yjk3NjEwNzdmNGI3MzU3YiA9ICQoJzxkaXYgaWQ9Imh0bWxfMTA3NDI2Y2ZjZjI3NDY2OWI5NzYxMDc3ZjRiNzM1N2IiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkJlZG9rIFJlc2Vydm9pciBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYmRmNGViYjgxYzlkNDNjZWJmYmUxN2IyMTg5NDcyNGYuc2V0Q29udGVudChodG1sXzEwNzQyNmNmY2YyNzQ2NjliOTc2MTA3N2Y0YjczNTdiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2JlYTI3MzBjZGFhNDRiMmI5NTU4ZTlhNjQzODZkOWY4LmJpbmRQb3B1cChwb3B1cF9iZGY0ZWJiODFjOWQ0M2NlYmZiZTE3YjIxODk0NzI0Zik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8xMzM5OGZiY2U5MDg0ZWQxYTNkOTIwYjE5ZmY3M2E5MCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjcxMzM2MDQ0NjYxODYyNywxMDMuODYyODU5NDM2Mjk0ODVdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzJkODBkNTEzM2EzNTRjMmViMjE1YmExMTAxNGYyZTUwID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2Q5M2E2NGIyMDM0ZjRmODZhNzAzMzA3MjA4OWQ1Y2FhID0gJCgnPGRpdiBpZD0iaHRtbF9kOTNhNjRiMjAzNGY0Zjg2YTcwMzMwNzIwODlkNWNhYSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+TWFyaW5hIFNvdXRoIFBpZXIgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzJkODBkNTEzM2EzNTRjMmViMjE1YmExMTAxNGYyZTUwLnNldENvbnRlbnQoaHRtbF9kOTNhNjRiMjAzNGY0Zjg2YTcwMzMwNzIwODlkNWNhYSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl8xMzM5OGZiY2U5MDg0ZWQxYTNkOTIwYjE5ZmY3M2E5MC5iaW5kUG9wdXAocG9wdXBfMmQ4MGQ1MTMzYTM1NGMyZWIyMTViYTExMDE0ZjJlNTApOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZDdlNjNlMzMxMzEwNDQ3ODgzMmY4ZWJiMzE0ZmY1MzYgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMxMjM1OTE3MzU0NDk5NCwxMDMuODU0MTc2ODA4MTkyMTRdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2JlZTI2ZDA0ZGE0NjRjNDhiMGVlMDZkODBmM2U5NjliID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzllYjhlN2VmOWE0NDQzM2E4NTBhOWZkZjhlM2RmNmIwID0gJCgnPGRpdiBpZD0iaHRtbF85ZWI4ZTdlZjlhNDQ0MzNhODUwYTlmZGY4ZTNkZjZiMCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+RmFycmVyIFBhcmsgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2JlZTI2ZDA0ZGE0NjRjNDhiMGVlMDZkODBmM2U5NjliLnNldENvbnRlbnQoaHRtbF85ZWI4ZTdlZjlhNDQ0MzNhODUwYTlmZGY4ZTNkZjZiMCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9kN2U2M2UzMzEzMTA0NDc4ODMyZjhlYmIzMTRmZjUzNi5iaW5kUG9wdXAocG9wdXBfYmVlMjZkMDRkYTQ2NGM0OGIwZWUwNmQ4MGYzZTk2OWIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfNmY3OGI3ZjcxNjEzNGZkY2IyMmM2ZDc0MWEwMzgyZjIgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM0MjgyNzY3MTA3MDc3MjUsMTAzLjg3OTc1ODU3Mjg3Mzg1XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81N2FlZTY1NzVhOWI0YzkyYWE5Njc5YjNmZTAxMjAxMyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83NDJhYmJjYjE1OTQ0MDdhYTIwM2E5ODdhZTBjNDNlMyA9ICQoJzxkaXYgaWQ9Imh0bWxfNzQyYWJiY2IxNTk0NDA3YWEyMDNhOTg3YWUwYzQzZTMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkJhcnRsZXkgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzU3YWVlNjU3NWE5YjRjOTJhYTk2NzliM2ZlMDEyMDEzLnNldENvbnRlbnQoaHRtbF83NDJhYmJjYjE1OTQ0MDdhYTIwM2E5ODdhZTBjNDNlMyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl82Zjc4YjdmNzE2MTM0ZmRjYjIyYzZkNzQxYTAzODJmMi5iaW5kUG9wdXAocG9wdXBfNTdhZWU2NTc1YTliNGM5MmFhOTY3OWIzZmUwMTIwMTMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZWViNjI3YzE1Zjc4NDM2M2E5NGYzMWMzZTFjZDdmYWYgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMwODM4MTk3MjYwMDEwMjUsMTAzLjg4ODY2MjI3MDU3MDU2XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF80NjFlYmJjOTQ5MDk0Mzg3YjI0ZTI1NDZiOTY0N2FjOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8zMTk2N2VkNDQ5NzY0OGIyYmQ3YWI5NGQ1MDc3YWEwMSA9ICQoJzxkaXYgaWQ9Imh0bWxfMzE5NjdlZDQ0OTc2NDhiMmJkN2FiOTRkNTA3N2FhMDEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkRha290YSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNDYxZWJiYzk0OTA5NDM4N2IyNGUyNTQ2Yjk2NDdhYzkuc2V0Q29udGVudChodG1sXzMxOTY3ZWQ0NDk3NjQ4YjJiZDdhYjk0ZDUwNzdhYTAxKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2VlYjYyN2MxNWY3ODQzNjNhOTRmMzFjM2UxY2Q3ZmFmLmJpbmRQb3B1cChwb3B1cF80NjFlYmJjOTQ5MDk0Mzg3YjI0ZTI1NDZiOTY0N2FjOSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8wYWFiNTMwNTY2YzE0YmYzOGJlOWE5MGQ5ZDAwZjAzMiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjc2MjEyODU2MDE4MTQzMiwxMDMuNzkxMzQ5OTc5ODM3OF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMmFlNTEwNmY0NDc4NDU0NmE2NzNkMTU3MGU2ZWMzYjEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNmJlNDFjZTJhNzZiNDRkN2I0ZmQ1NTE1YjA0M2RjZmMgPSAkKCc8ZGl2IGlkPSJodG1sXzZiZTQxY2UyYTc2YjQ0ZDdiNGZkNTUxNWIwNDNkY2ZjIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5QYXNpciBQYW5qYW5nIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yYWU1MTA2ZjQ0Nzg0NTQ2YTY3M2QxNTcwZTZlYzNiMS5zZXRDb250ZW50KGh0bWxfNmJlNDFjZTJhNzZiNDRkN2I0ZmQ1NTE1YjA0M2RjZmMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMGFhYjUzMDU2NmMxNGJmMzhiZTlhOTBkOWQwMGYwMzIuYmluZFBvcHVwKHBvcHVwXzJhZTUxMDZmNDQ3ODQ1NDZhNjczZDE1NzBlNmVjM2IxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzU3YmE2MTg0NGZlYTQ5OWNiNDMxOGYyODJiYjg3OGQ5ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yNzk3NjM4NzY1MTUzNjg3LDEwMy44Mzk1NzQ0MDc4ODM5Ml0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZTQyNGM4OTQxODk3NGI2ZGE4YzE5NjFhMDlmN2Y4NmEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMDBiZDBjYTk0MzU4NDZmNGE2OGUwMWI0N2M5MjZjNjUgPSAkKCc8ZGl2IGlkPSJodG1sXzAwYmQwY2E5NDM1ODQ2ZjRhNjhlMDFiNDdjOTI2YzY1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5PdXRyYW0gUGFyayBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZTQyNGM4OTQxODk3NGI2ZGE4YzE5NjFhMDlmN2Y4NmEuc2V0Q29udGVudChodG1sXzAwYmQwY2E5NDM1ODQ2ZjRhNjhlMDFiNDdjOTI2YzY1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzU3YmE2MTg0NGZlYTQ5OWNiNDMxOGYyODJiYjg3OGQ5LmJpbmRQb3B1cChwb3B1cF9lNDI0Yzg5NDE4OTc0YjZkYThjMTk2MWEwOWY3Zjg2YSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8wYzk0YWIxZDQ0MzM0ODJhYmYzM2E5YWJkMDkwZjBmYyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjg4Mzg1MzU3NTMzODQ4OSwxMDMuODQ2NTU0ODc1OTU4NTJdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzI2YzQxNDg1NTJkMDRiYTVhNDk1NWE2ZWY2MWRiMmUyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzg4YmY0NDVjYmYwNzRhMzU5Y2UxMTVkZGVkM2RhMzY3ID0gJCgnPGRpdiBpZD0iaHRtbF84OGJmNDQ1Y2JmMDc0YTM1OWNlMTE1ZGRlZDNkYTM2NyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+Q2xhcmtlIFF1YXkgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzI2YzQxNDg1NTJkMDRiYTVhNDk1NWE2ZWY2MWRiMmUyLnNldENvbnRlbnQoaHRtbF84OGJmNDQ1Y2JmMDc0YTM1OWNlMTE1ZGRlZDNkYTM2Nyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl8wYzk0YWIxZDQ0MzM0ODJhYmYzM2E5YWJkMDkwZjBmYy5iaW5kUG9wdXAocG9wdXBfMjZjNDE0ODU1MmQwNGJhNWE0OTU1YTZlZjYxZGIyZTIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZDExMzAwYzZlYTQ3NGJmNjgxZTA3ZmVjNzkwOWQ4YTAgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMyMjQyMzMxMjAzMjYyNjUsMTAzLjgxNjEzMTMxOTgwODg2XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8wNjQ3ZmIyNGM0ZGQ0NTIxYWUwNzk4YTk3ZjkxYjQyMCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8yZDQ1ZWI2YzIwNDc0NTUwYWU4OGI1ZDliMTBiZTY3ZSA9ICQoJzxkaXYgaWQ9Imh0bWxfMmQ0NWViNmMyMDQ3NDU1MGFlODhiNWQ5YjEwYmU2N2UiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkJvdGFuaWMgR2FyZGVucyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMDY0N2ZiMjRjNGRkNDUyMWFlMDc5OGE5N2Y5MWI0MjAuc2V0Q29udGVudChodG1sXzJkNDVlYjZjMjA0NzQ1NTBhZTg4YjVkOWIxMGJlNjdlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2QxMTMwMGM2ZWE0NzRiZjY4MWUwN2ZlYzc5MDlkOGEwLmJpbmRQb3B1cChwb3B1cF8wNjQ3ZmIyNGM0ZGQ0NTIxYWUwNzk4YTk3ZjkxYjQyMCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8yNzEwNDBkM2U1ZWE0ODdlOWU2MzJlY2M0MjU3NDU0NyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzQ4NzA2NTk2MzUxMjMzNSwxMDMuODM5NDIyNzk5MjM0OTJdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzFjMGYzY2Q4ZGUyZjQwODU4MTllZTI5ZTQ3YWE5NmQ4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2VhNTVjM2FjMWNkYzRlMDg5MWIzZDQzNjM3OGE0MGM2ID0gJCgnPGRpdiBpZD0iaHRtbF9lYTU1YzNhYzFjZGM0ZTA4OTFiM2Q0MzYzNzhhNDBjNiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+TWFyeW1vdW50IE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8xYzBmM2NkOGRlMmY0MDg1ODE5ZWUyOWU0N2FhOTZkOC5zZXRDb250ZW50KGh0bWxfZWE1NWMzYWMxY2RjNGUwODkxYjNkNDM2Mzc4YTQwYzYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMjcxMDQwZDNlNWVhNDg3ZTllNjMyZWNjNDI1NzQ1NDcuYmluZFBvcHVwKHBvcHVwXzFjMGYzY2Q4ZGUyZjQwODU4MTllZTI5ZTQ3YWE5NmQ4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2FjNzk1NzQwYzA2MTQ1M2E4ZjJkZWE2NzFhYjMwMzMyID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yODI1NDE0ODk5NzU4MzMsMTAzLjc4MTgxMDExODAxMTIxXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xMWRlM2E3ZTU2MWU0ZGE4YTk5Y2QwOTk4MTY5N2RkNSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF81NzdiZDhhNzE2MjE0NDcxYTI4MjU0ZGMwODg1NTYzMSA9ICQoJzxkaXYgaWQ9Imh0bWxfNTc3YmQ4YTcxNjIxNDQ3MWEyODI1NGRjMDg4NTU2MzEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkhhdyBQYXIgVmlsbGEgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzExZGUzYTdlNTYxZTRkYThhOTljZDA5OTgxNjk3ZGQ1LnNldENvbnRlbnQoaHRtbF81NzdiZDhhNzE2MjE0NDcxYTI4MjU0ZGMwODg1NTYzMSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9hYzc5NTc0MGMwNjE0NTNhOGYyZGVhNjcxYWIzMDMzMi5iaW5kUG9wdXAocG9wdXBfMTFkZTNhN2U1NjFlNGRhOGE5OWNkMDk5ODE2OTdkZDUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfOTBlNjAzMzFlOGMzNDg3YzhiODA3N2UyMDMxMGNkMWQgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM1MTYxMTUwNDM1ODc0NjksMTAzLjg2NDE1MTYwNDcwMzMxXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jNzNjNmZmZTIyNTk0N2Q0OGE0NGQ3M2EzZWVlOTg1YyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9mNmE4ZWFkNWM2OTY0MTVkOGNlNWFiOTMyYjcyZDZhZSA9ICQoJzxkaXYgaWQ9Imh0bWxfZjZhOGVhZDVjNjk2NDE1ZDhjZTVhYjkzMmI3MmQ2YWUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkxvcm9uZyBDaHVhbiBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYzczYzZmZmUyMjU5NDdkNDhhNDRkNzNhM2VlZTk4NWMuc2V0Q29udGVudChodG1sX2Y2YThlYWQ1YzY5NjQxNWQ4Y2U1YWI5MzJiNzJkNmFlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzkwZTYwMzMxZThjMzQ4N2M4YjgwNzdlMjAzMTBjZDFkLmJpbmRQb3B1cChwb3B1cF9jNzNjNmZmZTIyNTk0N2Q0OGE0NGQ3M2EzZWVlOTg1Yyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8yODEzZmQ5YWFiNWE0OGY4YTE3NTE0YWYwMmEwNmM3ZiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzA3MzU2MzUzNDI4NDM4NywxMDMuODYyODI4ODc0NzYzOF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOGI2YzcxMTYxZTMwNDI4ZGJiYmYzOWNiMGM3MWYxYmEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNzY0ZWU2YWFmODI5NGMwZjk1NmQwNTBlODhkNWU2NTEgPSAkKCc8ZGl2IGlkPSJodG1sXzc2NGVlNmFhZjgyOTRjMGY5NTZkMDUwZTg4ZDVlNjUxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5MYXZlbmRlciBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOGI2YzcxMTYxZTMwNDI4ZGJiYmYzOWNiMGM3MWYxYmEuc2V0Q29udGVudChodG1sXzc2NGVlNmFhZjgyOTRjMGY5NTZkMDUwZTg4ZDVlNjUxKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzI4MTNmZDlhYWI1YTQ4ZjhhMTc1MTRhZjAyYTA2YzdmLmJpbmRQb3B1cChwb3B1cF84YjZjNzExNjFlMzA0MjhkYmJiZjM5Y2IwYzcxZjFiYSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl82ZTI2NmQ5NzcwNWI0ZGQ2Yjc2MzFlOWI4ZTI4ZTg0MSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzM1NDMyNjU1NTU2MTU2LDEwMy44ODgxOTQ1Mjc3OTE0NF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOThjNjA4N2Q1YjNkNDYzYWE3MmE0OTBkYzliNzgxYWUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMDlkOGUwZDU3OWYzNDAxNWJlNzhiNDE4YmZjMDUyODUgPSAkKCc8ZGl2IGlkPSJodG1sXzA5ZDhlMGQ1NzlmMzQwMTViZTc4YjQxOGJmYzA1Mjg1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5UYWkgU2VuZyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOThjNjA4N2Q1YjNkNDYzYWE3MmE0OTBkYzliNzgxYWUuc2V0Q29udGVudChodG1sXzA5ZDhlMGQ1NzlmMzQwMTViZTc4YjQxOGJmYzA1Mjg1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzZlMjY2ZDk3NzA1YjRkZDZiNzYzMWU5YjhlMjhlODQxLmJpbmRQb3B1cChwb3B1cF85OGM2MDg3ZDViM2Q0NjNhYTcyYTQ5MGRjOWI3ODFhZSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl83NmI3N2ZmOTg0YWQ0NGUzYWNiNjZlNTVmMGU4ZDY1NCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzAzOTE1ODE3Mjk3Mzk0LDEwMy44NTI0MjEzNDczNjU5NV0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfODFlY2Y4N2E5MGI4NDk1NGE3ZDg3MzE5OGNkN2M4OWEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNTNiYzE2OThmMWQ4NGU3YmJiNjRjYmNmNzUwYmYwNGEgPSAkKCc8ZGl2IGlkPSJodG1sXzUzYmMxNjk4ZjFkODRlN2JiYjY0Y2JjZjc1MGJmMDRhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5Sb2Nob3IgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzgxZWNmODdhOTBiODQ5NTRhN2Q4NzMxOThjZDdjODlhLnNldENvbnRlbnQoaHRtbF81M2JjMTY5OGYxZDg0ZTdiYmI2NGNiY2Y3NTBiZjA0YSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl83NmI3N2ZmOTg0YWQ0NGUzYWNiNjZlNTVmMGU4ZDY1NC5iaW5kUG9wdXAocG9wdXBfODFlY2Y4N2E5MGI4NDk1NGE3ZDg3MzE5OGNkN2M4OWEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfNzRiMTJmNjVhZmEzNDM2Mzk1MjlhMTI2NGVlZmUxNTggPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMwNzE5NzU1NjI0MzkxNDksMTAzLjg0ODU4NDI5MjA0NDE4XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8zMjZiNjFjNzE0NmE0MjUzYmMzNmRjOTA4ODJlNmVkZiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jNWYyZjM3MzgzNTI0OTZlOWVjNGQ1OTgxNDFlYmNlOSA9ICQoJzxkaXYgaWQ9Imh0bWxfYzVmMmYzNzM4MzUyNDk2ZTllYzRkNTk4MTQxZWJjZTkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkxpdHRsZSBJbmRpYSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMzI2YjYxYzcxNDZhNDI1M2JjMzZkYzkwODgyZTZlZGYuc2V0Q29udGVudChodG1sX2M1ZjJmMzczODM1MjQ5NmU5ZWM0ZDU5ODE0MWViY2U5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzc0YjEyZjY1YWZhMzQzNjM5NTI5YTEyNjRlZWZlMTU4LmJpbmRQb3B1cChwb3B1cF8zMjZiNjFjNzE0NmE0MjUzYmMzNmRjOTA4ODJlNmVkZik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl82ZGUxOWM1MjVkODg0YzU0YTg0MGE0NDE0NGUwNzY2YiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzIwMDY0ODkwMzM5NTg4LDEwMy44MjYwMjQwNjg5MTI3N10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDEzYjM4NjQwMTZkNGYzYzk0ZDA5ZDJhMmVhN2MyODEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNWUwMzllZDM5NTUwNDc2OGJhYTViNmM1MTBiZmJhZmQgPSAkKCc8ZGl2IGlkPSJodG1sXzVlMDM5ZWQzOTU1MDQ3NjhiYWE1YjZjNTEwYmZiYWZkIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TdGV2ZW5zIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80MTNiMzg2NDAxNmQ0ZjNjOTRkMDlkMmEyZWE3YzI4MS5zZXRDb250ZW50KGh0bWxfNWUwMzllZDM5NTUwNDc2OGJhYTViNmM1MTBiZmJhZmQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfNmRlMTljNTI1ZDg4NGM1NGE4NDBhNDQxNDRlMDc2NmIuYmluZFBvcHVwKHBvcHVwXzQxM2IzODY0MDE2ZDRmM2M5NGQwOWQyYTJlYTdjMjgxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2Y4MGQ5MjRhZjM3MDQ5ZDI4NjMxZTY5NTZhNDcxNGZjID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNjkzNjkxNjQzOTc5NDA5LDEwMy43NjQ2OTQwODkwMjUwM10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNGM0MTU4YjM3MTdmNGM2M2I2NGQzNzNkMWY5MzkxZWQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYzQyYjRhZjFhMmRlNDA3YmFlNzUwYjNjMzg3ZjdjNzEgPSAkKCc8ZGl2IGlkPSJodG1sX2M0MmI0YWYxYTJkZTQwN2JhZTc1MGIzYzM4N2Y3YzcxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5DYXNoZXcgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzRjNDE1OGIzNzE3ZjRjNjNiNjRkMzczZDFmOTM5MWVkLnNldENvbnRlbnQoaHRtbF9jNDJiNGFmMWEyZGU0MDdiYWU3NTBiM2MzODdmN2M3MSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9mODBkOTI0YWYzNzA0OWQyODYzMWU2OTU2YTQ3MTRmYy5iaW5kUG9wdXAocG9wdXBfNGM0MTU4YjM3MTdmNGM2M2I2NGQzNzNkMWY5MzkxZWQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfOTIzYmU2ZTM2YjNmNGVlMjkyMjRiYmFiZjIwOTYxODMgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMzMDc4NTcyMDY5NzAyNTcsMTAzLjc5NzI0NTk4MzM0MTYzXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xOWRkNzFlYmU2OTg0NDExOGYwZDY4MzU3YjM5YWY4MSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82MDEyODM1ZjU5MjA0ZDJkOTUxYTFiMWYyMTYwNWVmOSA9ICQoJzxkaXYgaWQ9Imh0bWxfNjAxMjgzNWY1OTIwNGQyZDk1MWExYjFmMjE2MDVlZjkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNpeHRoIEF2ZW51ZSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMTlkZDcxZWJlNjk4NDQxMThmMGQ2ODM1N2IzOWFmODEuc2V0Q29udGVudChodG1sXzYwMTI4MzVmNTkyMDRkMmQ5NTFhMWIxZjIxNjA1ZWY5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzkyM2JlNmUzNmIzZjRlZTI5MjI0YmJhYmYyMDk2MTgzLmJpbmRQb3B1cChwb3B1cF8xOWRkNzFlYmU2OTg0NDExOGYwZDY4MzU3YjM5YWY4MSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl9hNDk3M2UwMDgxZTQ0M2M1YTMzMjU2NTMwOGEzZDQ3YiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzMyNjI4MzIwNDU2MTQxLDEwMy44NDc1MDE0MzEwMDMxNF0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNmNhOGQ5YTY0NTMyNDY3MDg3NDY5YTY5ZDg1ZWNlMjMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNGNmNDQ4NTYyNzFhNDYwNDkzYzU4Y2I2NWJkYTQ4MzYgPSAkKCc8ZGl2IGlkPSJodG1sXzRjZjQ0ODU2MjcxYTQ2MDQ5M2M1OGNiNjViZGE0ODM2IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5Ub2EgUGF5b2ggTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzZjYThkOWE2NDUzMjQ2NzA4NzQ2OWE2OWQ4NWVjZTIzLnNldENvbnRlbnQoaHRtbF80Y2Y0NDg1NjI3MWE0NjA0OTNjNThjYjY1YmRhNDgzNik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl9hNDk3M2UwMDgxZTQ0M2M1YTMzMjU2NTMwOGEzZDQ3Yi5iaW5kUG9wdXAocG9wdXBfNmNhOGQ5YTY0NTMyNDY3MDg3NDY5YTY5ZDg1ZWNlMjMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfOTU5YjFiZjFhNjU3NGI0MzhmYTEyYzRmNWNkMzRiMjggPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjQyNzI1OTMxMjA0NDg0MiwxMDMuNzkzODUwNTU5MTIxOTldLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2MxNWVmYTRlZmMzYzQxNDA4YjY4NThlZjJhODU2MGFiID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2E3MmVlNDI2NTg1NjQ2ZTNhNTZjYzgxN2JlMTYwZGYzID0gJCgnPGRpdiBpZD0iaHRtbF9hNzJlZTQyNjU4NTY0NmUzYTU2Y2M4MTdiZTE2MGRmMyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+V29vZGxhbmRzIFNvdXRoIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9jMTVlZmE0ZWZjM2M0MTQwOGI2ODU4ZWYyYTg1NjBhYi5zZXRDb250ZW50KGh0bWxfYTcyZWU0MjY1ODU2NDZlM2E1NmNjODE3YmUxNjBkZjMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfOTU5YjFiZjFhNjU3NGI0MzhmYTEyYzRmNWNkMzRiMjguYmluZFBvcHVwKHBvcHVwX2MxNWVmYTRlZmMzYzQxNDA4YjY4NThlZjJhODU2MGFiKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzdkZTJjOWFkYTgwYTQ0OTU4NzQ1YmFmZTU5YjNkNzc2ID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMzEzNzg4NTc5NTIzMzczLDEwMy44NjkwNTUzMzE5MjU5Ml0sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMGYyYmJmOTI4MWZkNGZmY2JlZTM0MGM3ZWFmMDEwZjIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZTY0YWI0MjlmNTcwNDY0OGI1MjAyMDNkNjQyZDdlZWUgPSAkKCc8ZGl2IGlkPSJodG1sX2U2NGFiNDI5ZjU3MDQ2NDhiNTIwMjAzZDY0MmQ3ZWVlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5Qb3RvbmcgUGFzaXIgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzBmMmJiZjkyODFmZDRmZmNiZWUzNDBjN2VhZjAxMGYyLnNldENvbnRlbnQoaHRtbF9lNjRhYjQyOWY1NzA0NjQ4YjUyMDIwM2Q2NDJkN2VlZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl83ZGUyYzlhZGE4MGE0NDk1ODc0NWJhZmU1OWIzZDc3Ni5iaW5kUG9wdXAocG9wdXBfMGYyYmJmOTI4MWZkNGZmY2JlZTM0MGM3ZWFmMDEwZjIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfNGViYzdlZWZlOTMxNDJiYThhYjkxNTE1NDNiNTlhZTIgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM3MTI5MTYyNTAxMjAyMDIsMTAzLjg5MjM4MDE4NTU0NDI4XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF82ZTVjMGJiZjYzMDA0NmUzOTA1Y2Q3OGQ5ZGYyMDU3NiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83YjdlMDRiOTcxYWM0NjZkYWZjMmYzMmU4MGM5NGFmMiA9ICQoJzxkaXYgaWQ9Imh0bWxfN2I3ZTA0Yjk3MWFjNDY2ZGFmYzJmMzJlODBjOTRhZjIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkhvdWdhbmcgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzZlNWMwYmJmNjMwMDQ2ZTM5MDVjZDc4ZDlkZjIwNTc2LnNldENvbnRlbnQoaHRtbF83YjdlMDRiOTcxYWM0NjZkYWZjMmYzMmU4MGM5NGFmMik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl80ZWJjN2VlZmU5MzE0MmJhOGFiOTE1MTU0M2I1OWFlMi5iaW5kUG9wdXAocG9wdXBfNmU1YzBiYmY2MzAwNDZlMzkwNWNkNzhkOWRmMjA1NzYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfODk4ZjViNDRiYWU3NDI1NDk2MDVmZWVkOWJmNWFiMzYgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjMxOTM5NTAzOTM1NTk4NDYsMTAzLjg2MTY4NjE2ODUyNzM3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iYTgwNTllM2M0MDA0YzRiYTE1MGIyYWNkODkwMjM1NCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83NWRmNmM4NGU0YmQ0ZjQyYjg2NTU1ZDYwOWI1NWRmYiA9ICQoJzxkaXYgaWQ9Imh0bWxfNzVkZjZjODRlNGJkNGY0MmI4NjU1NWQ2MDliNTVkZmIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkJvb24gS2VuZyBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYmE4MDU5ZTNjNDAwNGM0YmExNTBiMmFjZDg5MDIzNTQuc2V0Q29udGVudChodG1sXzc1ZGY2Yzg0ZTRiZDRmNDJiODY1NTVkNjA5YjU1ZGZiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzg5OGY1YjQ0YmFlNzQyNTQ5NjA1ZmVlZDliZjVhYjM2LmJpbmRQb3B1cChwb3B1cF9iYTgwNTllM2M0MDA0YzRiYTE1MGIyYWNkODkwMjM1NCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8yMmFkYjU0Mzc4MzI0YWJkYjVlMGJjNWEzZDc3ODk3YyA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzQxNzM2ODE3MjI3ODQyOCwxMDMuOTYxNDcxNDU1MTMzODVdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2ZkMzhiMTYwMjFhZjQ1MTdiMTlmYTliZjRmZTQ2MjgxID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzFlOWVjZWU0YTQxMjQzMWY4OGQxZTg4MWQ0Yjg4OTI1ID0gJCgnPGRpdiBpZD0iaHRtbF8xZTllY2VlNGE0MTI0MzFmODhkMWU4ODFkNGI4ODkyNSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+VXBwZXIgQ2hhbmdpIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9mZDM4YjE2MDIxYWY0NTE3YjE5ZmE5YmY0ZmU0NjI4MS5zZXRDb250ZW50KGh0bWxfMWU5ZWNlZTRhNDEyNDMxZjg4ZDFlODgxZDRiODg5MjUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfMjJhZGI1NDM3ODMyNGFiZGI1ZTBiYzVhM2Q3Nzg5N2MuYmluZFBvcHVwKHBvcHVwX2ZkMzhiMTYwMjFhZjQ1MTdiMTlmYTliZjRmZTQ2MjgxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2ZjOWQ1MTRjMGQ2NjQyMjg4ZGQzM2RlM2M1ZTk0NmNiID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zNTczMTM4Nzg2NjY0ODU2LDEwMy45ODgzNjQzMjQ4MDkyN10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZDAxZTAyNmE2MTQ1NDczMDhmMTRjMGY0YjZkMmQyZjYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNjFjZjFiNDFmYWIyNDNkOThmYjhhYzIyNDkzYTU5NTAgPSAkKCc8ZGl2IGlkPSJodG1sXzYxY2YxYjQxZmFiMjQzZDk4ZmI4YWMyMjQ5M2E1OTUwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5DaGFuZ2kgQWlycG9ydCBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZDAxZTAyNmE2MTQ1NDczMDhmMTRjMGY0YjZkMmQyZjYuc2V0Q29udGVudChodG1sXzYxY2YxYjQxZmFiMjQzZDk4ZmI4YWMyMjQ5M2E1OTUwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyX2ZjOWQ1MTRjMGQ2NjQyMjg4ZGQzM2RlM2M1ZTk0NmNiLmJpbmRQb3B1cChwb3B1cF9kMDFlMDI2YTYxNDU0NzMwOGYxNGMwZjRiNmQyZDJmNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8xNjFkMTdmMTg0YTM0YmYyOWNiMjYwNDY4ZGRiNThiMCA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMzQwNDcxMDE3Nzg0MDc4MiwxMDMuODQ2Nzk3NTEyMDc1NDddLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzUzZTAzZjEzNTYyMDRkYWRiZGY0YTMzZjhhZjY5NWU1ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzM0MjgxYzNiOWRiYjRjOThiZDNkMGFjMTU4ZTZiNzdlID0gJCgnPGRpdiBpZD0iaHRtbF8zNDI4MWMzYjlkYmI0Yzk4YmQzZDBhYzE1OGU2Yjc3ZSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+QnJhZGRlbGwgTXJ0IFN0YXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzUzZTAzZjEzNTYyMDRkYWRiZGY0YTMzZjhhZjY5NWU1LnNldENvbnRlbnQoaHRtbF8zNDI4MWMzYjlkYmI0Yzk4YmQzZDBhYzE1OGU2Yjc3ZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgY2lyY2xlX21hcmtlcl8xNjFkMTdmMTg0YTM0YmYyOWNiMjYwNDY4ZGRiNThiMC5iaW5kUG9wdXAocG9wdXBfNTNlMDNmMTM1NjIwNGRhZGJkZjRhMzNmOGFmNjk1ZTUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGNpcmNsZV9tYXJrZXJfZTIyNjliMDEyM2FhNGFmZGEyYzhhOGVmZTViN2NlNjYgPSBMLmNpcmNsZU1hcmtlcigKICAgICAgICAgICAgICAgIFsxLjM4Mjg3NzE5MTQ4MDc3OTQsMTAzLjg5MzEyMTIzMTQxNjUyXSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF80ZGZjM2ZhZWM5ZWY0OTA5Yjk0YjkwN2Y0MWJkYzQ2ZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80OTY0ZDUxODdlZmM0ZjYxOWQ2MGQ2NjE3MmMxOGI1OCA9ICQoJzxkaXYgaWQ9Imh0bWxfNDk2NGQ1MTg3ZWZjNGY2MTlkNjBkNjYxNzJjMThiNTgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkJ1YW5na29rIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80ZGZjM2ZhZWM5ZWY0OTA5Yjk0YjkwN2Y0MWJkYzQ2ZS5zZXRDb250ZW50KGh0bWxfNDk2NGQ1MTg3ZWZjNGY2MTlkNjBkNjYxNzJjMThiNTgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZTIyNjliMDEyM2FhNGFmZGEyYzhhOGVmZTViN2NlNjYuYmluZFBvcHVwKHBvcHVwXzRkZmMzZmFlYzllZjQ5MDliOTRiOTA3ZjQxYmRjNDZlKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyX2Q1ZTA1MzBiYjNmMzRlZGZhN2VkMjI1N2M2NTUzZTFjID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4yODQxMjQ5NDM5MDU5MDY2LDEwMy44NTE0NjEzNzg3NTI5M10sCiAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICIjMzM4OGZmIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogIiNjY2NjY2MiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm9wYWNpdHkiOiAxLjAsCiAgInJhZGl1cyI6IDMsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDMKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfYjZmZWY4NDEyYmQyNDhmYzg5ZTgwYzRkYjkxMTFhNGMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfM2ZjMWQxM2NiMDlkNGU4NDk0YjFmNDdmY2MwYjdjNDMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNzdlYjY2NTc1ZjQ2NGFmNTk4MDM1YTFhYTczMjc3MTQgPSAkKCc8ZGl2IGlkPSJodG1sXzc3ZWI2NjU3NWY0NjRhZjU5ODAzNWExYWE3MzI3NzE0IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5SYWZmbGVzIFBsYWNlIE1ydCBTdGF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8zZmMxZDEzY2IwOWQ0ZTg0OTRiMWY0N2ZjYzBiN2M0My5zZXRDb250ZW50KGh0bWxfNzdlYjY2NTc1ZjQ2NGFmNTk4MDM1YTFhYTczMjc3MTQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIGNpcmNsZV9tYXJrZXJfZDVlMDUzMGJiM2YzNGVkZmE3ZWQyMjU3YzY1NTNlMWMuYmluZFBvcHVwKHBvcHVwXzNmYzFkMTNjYjA5ZDRlODQ5NGIxZjQ3ZmNjMGI3YzQzKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBjaXJjbGVfbWFya2VyXzU2YWRmMDViYzYxMDRkYWQ5MjkyNWJmM2QxNGQzYjUwID0gTC5jaXJjbGVNYXJrZXIoCiAgICAgICAgICAgICAgICBbMS4zMTE4MzQxMjM3MzUwODQsMTAzLjc5NjE5MTQ5MDU0MjQ3XSwKICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiMzMzg4ZmYiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI2NjY2NjYyIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAib3BhY2l0eSI6IDEuMCwKICAicmFkaXVzIjogMywKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMwp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9iNmZlZjg0MTJiZDI0OGZjODllODBjNGRiOTExMWE0Yyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF82Nzg4OWQ1MDFiNzk0NjRlOWNjOWM4NTVlNjc3MzMyMCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80YWMwNzdmZWY0NDU0MTYwYjk4MjJlOTg2OGU0OWM4YyA9ICQoJzxkaXYgaWQ9Imh0bWxfNGFjMDc3ZmVmNDQ1NDE2MGI5ODIyZTk4NjhlNDljOGMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPkhvbGxhbmQgVmlsbGFnZSBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNjc4ODlkNTAxYjc5NDY0ZTljYzljODU1ZTY3NzMzMjAuc2V0Q29udGVudChodG1sXzRhYzA3N2ZlZjQ0NTQxNjBiOTgyMmU5ODY4ZTQ5YzhjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzU2YWRmMDViYzYxMDRkYWQ5MjkyNWJmM2QxNGQzYjUwLmJpbmRQb3B1cChwb3B1cF82Nzg4OWQ1MDFiNzk0NjRlOWNjOWM4NTVlNjc3MzMyMCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl8wM2YyN2E3YWEwOTk0MjdjYjFjNjFmY2MxODFmNjBlZSA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjcwNzUyNTQ0NzkxMjM2NywxMDMuODA5NzQ4MzAzNTA5NjhdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzczZTc2ZTMzZjg4MjQ3NGJhNTQxM2I0ZWRlZGVlYjdmID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2FiNjA4NTU5MjY2ODQxYWJiMjZjZTRmNTE4Y2NiODEyID0gJCgnPGRpdiBpZD0iaHRtbF9hYjYwODU1OTI2Njg0MWFiYjI2Y2U0ZjUxOGNjYjgxMiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+VGVsb2sgQmxhbmdhaCBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNzNlNzZlMzNmODgyNDc0YmE1NDEzYjRlZGVkZWViN2Yuc2V0Q29udGVudChodG1sX2FiNjA4NTU5MjY2ODQxYWJiMjZjZTRmNTE4Y2NiODEyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzAzZjI3YTdhYTA5OTQyN2NiMWM2MWZjYzE4MWY2MGVlLmJpbmRQb3B1cChwb3B1cF83M2U3NmUzM2Y4ODI0NzRiYTU0MTNiNGVkZWRlZWI3Zik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgY2lyY2xlX21hcmtlcl82MmU0MjljMWYxNjI0ZTMwYjc3MDcyNWMwYmY3YmNkYiA9IEwuY2lyY2xlTWFya2VyKAogICAgICAgICAgICAgICAgWzEuMjgyMjg4ODY5NTI0OTM2NCwxMDMuODQ4MzAyNDcyODU1NThdLAogICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiIzMzODhmZiIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICIjY2NjY2NjIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJvcGFjaXR5IjogMS4wLAogICJyYWRpdXMiOiAzLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiAzCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwX2I2ZmVmODQxMmJkMjQ4ZmM4OWU4MGM0ZGI5MTExYTRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2VhMzJiZjNjMTYxYjRkM2U4YmIwNDc4ZDExMjdmZTk3ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzBiYmUwZTQ4ZjVhYTQ3NTI5OWJjNmQ1MzY3ZmM3NzdmID0gJCgnPGRpdiBpZD0iaHRtbF8wYmJlMGU0OGY1YWE0NzUyOTliYzZkNTM2N2ZjNzc3ZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+VGVsb2sgQXllciBNcnQgU3RhdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZWEzMmJmM2MxNjFiNGQzZThiYjA0NzhkMTEyN2ZlOTcuc2V0Q29udGVudChodG1sXzBiYmUwZTQ4ZjVhYTQ3NTI5OWJjNmQ1MzY3ZmM3NzdmKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBjaXJjbGVfbWFya2VyXzYyZTQyOWMxZjE2MjRlMzBiNzcwNzI1YzBiZjdiY2RiLmJpbmRQb3B1cChwb3B1cF9lYTMyYmYzYzE2MWI0ZDNlOGJiMDQ3OGQxMTI3ZmU5Nyk7CgogICAgICAgICAgICAKICAgICAgICAKPC9zY3JpcHQ+" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>
