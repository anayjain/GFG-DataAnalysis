# Project Name: GeeksforGeeks Data Analysis
Scrape the Geeksforgeeks youtube channel videos of the past 6 months' dataset

## Tasks & Questions:
1. Number of videos in the past 6 months from the start date. Must mention the dates in the solution.
2. Create a pandas data frame with columns name as videos title, views, Length of videos, and videos link
3. Name the most viewed topics in the past 6 months.
4. Name the topics with the highest video length.
5. Make a comparison between the number of views and video length using a Graph.

Dataset: https://www.youtube.com/@GeeksforGeeksVideos/videos

### Enabling YouTube API
To enable YouTube Data API, you should follow below steps:

1. Go to Google's API Console and create a project, or use an existing one. https://console.cloud.google.com/
2. In the library panel, search for YouTube Data API v3, click on it and click Enable.
3. In the credentials panel, click on Create Credentials, and choose API key.
4. You'll see a window with the API key. Make sure to copy and save the API key generated, we will use it later.

### Install required libraries


```python
!pip3 install --upgrade google-api-python-client #install google api client for api access
!pip3 install utcnow
!pip3 install itables
!pip3 install isoduration
```

### Download Channel Data and Id


```python
from googleapiclient.discovery import build

from utcnow import utcnow
from datetime import datetime, timedelta
from dateutil.relativedelta import *
import math

import json
import pandas as pd

from itables import show
from tabulate import tabulate

import plotly.express as px

from isoduration import parse_duration

```


```python
def get_channel_videos(youtube, **kwargs):
    return youtube.search().list(
        **kwargs
    ).execute()

def search(youtube, **kwargs):
    return youtube.search().list(
        part="snippet",
        **kwargs
    ).execute()

def get_video_details(youtube, **kwargs):
    return youtube.videos().list(
        part="snippet,contentDetails,statistics",
        **kwargs
    ).execute()
```


```python
# API information
api_service_name = "youtube"
api_version = "v3"
# API key
DEVELOPER_KEY = "AIzaSyDOe4w7EHaeRfqzrG7JyadEuwFowxM6YXs"
# API client
youtube = build(
    api_service_name, api_version, developerKey = DEVELOPER_KEY)
```


```python
channel_url = "https://www.youtube.com/@GeeksforGeeksVideos"
# get the channel name from the URL
name = channel_url.split("/")[-1]
response_id = search(youtube, q=name, maxResults=1)
items = response_id.get("items")
if items:
    channel_id = items[0]["snippet"]["channelId"]
print(channel_id)
```

    UC0RhatS1pyxInC00YKjjBqQ


### Extract All Video data based on Channel Id


```python
# First get current datatime timsetamp
current_time = datetime.now()

# Subtract 6 months from current datatime timestamp
current_time_RFC3339 = utcnow.get(current_time)
after = current_time - relativedelta(months=+6)
after_RFC3339 = utcnow.get(after)

#Run search for all channel video published after datetime provided
params = {
        'part': 'snippet',
        'q': '',
        'channelId': channel_id,
        'publishedAfter': after_RFC3339,
        'type': 'video',
    }
res = get_channel_videos(youtube, **params)
# get total results and check how many pages of videos are listed
tot = res["pageInfo"]['totalResults']
n_pages = math.ceil(tot / 50)
n_videos = 0
video_data = []
video_url = []
next_page_token = None
# Get all results through pagination
for i in range(n_pages):
    params = {
        'part': 'snippet',
        'q': '',
        'channelId': channel_id,
        'publishedAfter': after_RFC3339,
        'maxResults': 50, # maximum permitted result per page
        'type': 'video',
    }
    # this is to get the next page and save the token
    if next_page_token:
        params['pageToken'] = next_page_token
    res = get_channel_videos(youtube, **params)
    channel_videos = res.get("items")
    for v in channel_videos:
      n_videos += 1
      video_id = v["id"]["videoId"]
      # easily construct video URL by its ID
      url = f"https://www.youtube.com/watch?v={video_id}"
      # check for unique videos
      if url not in video_url:
        video_url.append(url)
        video_response = get_video_details(youtube, id=video_id)
        video_data.append(video_response.get("items")[0])
    if "nextPageToken" in res:
        next_page_token = res["nextPageToken"]
```

### Number of videos in the past 6 months from the start date.
(Must mention the dates in the solution.)


```python
totalResults = n_videos
print("The total number of videos in past 6 months from {} is {}.".format(after_RFC3339,totalResults))

```

    The total number of videos in past 6 months from 2023-02-07T17:42:17.567678Z is 202.


### Create a pandas data frame with columns name as videos title, views, Length of videos, and videos link


```python
# Save entire video data in dataframe
df = pd.DataFrame()
for i in range(len(video_data)):
  snippet = video_data[i]['snippet']
  content = video_data[i]['contentDetails']
  stats = video_data[i]['statistics']
  snippet.update(content)
  snippet.update(stats)
  df = pd.concat([df, pd.DataFrame.from_records([snippet])], ignore_index=True)
df['video_link'] = video_url
# Save a copy of raw data
raw_table = df
# Remove unnecessary columns
df = df.drop(['publishedAt','channelId','description','thumbnails','tags','categoryId'],axis=1)
df = df.drop(['liveBroadcastContent','defaultLanguage','caption','licensedContent','contentRating','projection'],axis=1)
df = df.drop(['localized','defaultAudioLanguage','dimension','definition'],axis=1)
# show(df.head(5))
tabulate(df.head(5), headers = 'keys', tablefmt = 'html')
```




<table>
<thead>
<tr><th style="text-align: right;">  </th><th>title                                                          </th><th>channelTitle  </th><th>duration  </th><th style="text-align: right;">  viewCount</th><th style="text-align: right;">  likeCount</th><th style="text-align: right;">  favoriteCount</th><th style="text-align: right;">  commentCount</th><th style="text-align: right;">  regionRestriction</th><th>video_link                                 </th></tr>
</thead>
<tbody>
<tr><td style="text-align: right;"> 0</td><td>Create Your Own Apps Today | GeeksforGeeks                     </td><td>GeeksforGeeks </td><td>PT38S     </td><td style="text-align: right;">      34923</td><td style="text-align: right;">         89</td><td style="text-align: right;">              0</td><td style="text-align: right;">             1</td><td style="text-align: right;">                nan</td><td>https://www.youtube.com/watch?v=LycNCWC3g18</td></tr>
<tr><td style="text-align: right;"> 1</td><td>Can you solve this puzzle? | Give your answers in comments 👇🏻</td><td>GeeksforGeeks </td><td>PT33S     </td><td style="text-align: right;">       2411</td><td style="text-align: right;">        119</td><td style="text-align: right;">              0</td><td style="text-align: right;">            11</td><td style="text-align: right;">                nan</td><td>https://www.youtube.com/watch?v=CGZkoNtQl_U</td></tr>
<tr><td style="text-align: right;"> 2</td><td>First Ever KBC at our Offline Classes | GeeksforGeeks          </td><td>GeeksforGeeks </td><td>PT37S     </td><td style="text-align: right;">        598</td><td style="text-align: right;">         19</td><td style="text-align: right;">              0</td><td style="text-align: right;">             0</td><td style="text-align: right;">                nan</td><td>https://www.youtube.com/watch?v=chYg2EAKW2A</td></tr>
<tr><td style="text-align: right;"> 3</td><td>Caution : 100% Relatable                                       </td><td>GeeksforGeeks </td><td>PT25S     </td><td style="text-align: right;">       5863</td><td style="text-align: right;">        277</td><td style="text-align: right;">              0</td><td style="text-align: right;">             2</td><td style="text-align: right;">                nan</td><td>https://www.youtube.com/watch?v=0NoD0Y_mAy4</td></tr>
<tr><td style="text-align: right;"> 4</td><td>😡😤                                                           </td><td>GeeksforGeeks </td><td>PT11S     </td><td style="text-align: right;">       4155</td><td style="text-align: right;">        164</td><td style="text-align: right;">              0</td><td style="text-align: right;">             2</td><td style="text-align: right;">                nan</td><td>https://www.youtube.com/watch?v=24G2og7huNY</td></tr>
</tbody>
</table>



### Most viewed topics in the past 6 months.


```python
# Convert viewCount from string to integer
df['viewCount'] = df['viewCount'].astype('int')
# Sort topics by number of views
most_viewed = df.sort_values(by=['viewCount'],ascending=False)
print("Most viewed topics in the past 6 months ->")
# show(most_viewed[['title', 'viewCount', 'video_link']].head(5))
tabulate(most_viewed[['title', 'viewCount', 'video_link']].head(5), headers = 'keys', tablefmt = 'html')
```

    Most viewed topics in the past 6 months ->





<table>
<thead>
<tr><th style="text-align: right;">  </th><th>title                                                                                     </th><th style="text-align: right;">  viewCount</th><th>video_link                                 </th></tr>
</thead>
<tbody>
<tr><td style="text-align: right;">28</td><td>Learn System Design with GeeksforGeeks                                                    </td><td style="text-align: right;">     118339</td><td>https://www.youtube.com/watch?v=XQEZ07JhVuA</td></tr>
<tr><td style="text-align: right;">20</td><td>GeeksforGeeks Classroom Program | Now in Noida and Gurugram!                              </td><td style="text-align: right;">     111314</td><td>https://www.youtube.com/watch?v=16D2cuRy5JY</td></tr>
<tr><td style="text-align: right;">42</td><td>Free Summer Offline Classes on Python Programing | For Students Aged 14-21 | GeeksforGeeks</td><td style="text-align: right;">     106937</td><td>https://www.youtube.com/watch?v=OOLXHwZzHfM</td></tr>
<tr><td style="text-align: right;">39</td><td>Full Stack Development | LIVE Classes | GeeksforGeeks                                     </td><td style="text-align: right;">      91939</td><td>https://www.youtube.com/watch?v=cBfC9HLR9Qk</td></tr>
<tr><td style="text-align: right;">35</td><td>Job Fair for Students | Till 25th May Only | GeeksforGeeks                                </td><td style="text-align: right;">      82854</td><td>https://www.youtube.com/watch?v=1GEegOb3fHE</td></tr>
</tbody>
</table>




```python
print("The most viewed topic in past 6 months is {} with {} views.".format(list(most_viewed['title'])[0],list(most_viewed['viewCount'])[0]))
```

    The most viewed topic in past 6 months is Learn System Design with GeeksforGeeks with 118339 views.


### Topics with the highest video length


```python
# create function to parse ISO-8601 duration and calculate the total seconds
def calculate_duration_sec(s):
  duration = parse_duration(s)
  tot_sec = (int(duration.time.hours) * 60 * 60) + (int(duration.time.minutes) * 60) + int(duration.time.seconds)
  return tot_sec

#Append a new column duration in seconds to table
duration = list(df['duration'])
duration_sec = []
for i in duration:
  duration_sec.append(calculate_duration_sec(i))
df.insert(3,'duration_in_seconds', duration_sec)

most_duration = df.sort_values(by=['duration_in_seconds'],ascending=False)
print("Longest duration topics in the past 6 months ->")
# show(most_duration[['title', 'duration', 'duration_in_seconds', 'video_link']].head(5))
tabulate(most_duration[['title', 'duration', 'duration_in_seconds', 'video_link']].head(5), headers = 'keys', tablefmt = 'html')
```

    Longest duration topics in the past 6 months ->





<table>
<thead>
<tr><th style="text-align: right;">   </th><th>title                                                   </th><th>duration  </th><th style="text-align: right;">  duration_in_seconds</th><th>video_link                                 </th></tr>
</thead>
<tbody>
<tr><td style="text-align: right;">138</td><td>SDE Preparation in 3 hours                              </td><td>PT2H55M27S</td><td style="text-align: right;">                10527</td><td>https://www.youtube.com/watch?v=ftDoBLp-OfU</td></tr>
<tr><td style="text-align: right;"> 98</td><td>CodeCamp Day 2 | Exploring Arrays and Problem Solving   </td><td>PT2H29M20S</td><td style="text-align: right;">                 8960</td><td>https://www.youtube.com/watch?v=OMHeYpQCCPE</td></tr>
<tr><td style="text-align: right;">187</td><td>CodeCamp Day 13 | Discovering Graph Traversal Algorithms</td><td>PT2H14M24S</td><td style="text-align: right;">                 8064</td><td>https://www.youtube.com/watch?v=hgkjJD5hb5g</td></tr>
<tr><td style="text-align: right;">197</td><td>CodeCamp Day 19 | Exploring Advanced Topics in DSA      </td><td>PT2H13M39S</td><td style="text-align: right;">                 8019</td><td>https://www.youtube.com/watch?v=kLZGFHK_bIc</td></tr>
<tr><td style="text-align: right;">134</td><td>CodeCamp Day 11 | Journey into Binary Trees             </td><td>PT2H10M41S</td><td style="text-align: right;">                 7841</td><td>https://www.youtube.com/watch?v=U1UKjsA4jNg</td></tr>
</tbody>
</table>



### Scatter Plot Graph of number of views v/s video length


```python
data = df

df.to_csv('youtube_video_6months.csv', index=False)

data['duration_in_millisec'] = data['duration_in_seconds'] * 1000
```


```python
fig = px.scatter(data, x="duration_in_millisec", y="viewCount",
                 trendline="rolling", trendline_options=dict(window=5, win_type="gaussian", function_args=dict(std=2)), trendline_color_override="mediumpurple",
                  title="No. of Views V/S Length of Videos", hover_data=['title'], labels={})
fig.update_traces(marker={
    'size': 7,
    'color': 'royalblue',
    'symbol': 'circle-open'})
fig.update_layout(showlegend=True)

fig.update_layout(
    legend=dict(
        x=0.01,
        y=.98,
        traceorder="normal",
        font=dict(
            family="sans-serif",
            size=12,
            color="Black"
        ),
        bgcolor="LightSteelBlue",
        bordercolor="dimgray",
        borderwidth=2
    ))

# fig.show()
```

![newplot.png](https://github.com/anayjain/GFG-DataAnalysis/blob/main/newplot.png)

From the graph above you can see that no. of views is extremely high when the duration of videos is less. As the duration increases there is steep decline in viewer count until it reaches a stable value wherein even 10million millisec duration videos get a low viewer count.
