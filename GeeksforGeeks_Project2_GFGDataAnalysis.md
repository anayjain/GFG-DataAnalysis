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

    The total number of videos in past 6 months from 2023-02-07T16:20:24.839240Z is 202.


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
show(df.head(5))
# tabulate(df.head(5), headers = 'keys', tablefmt = 'html')
```


<style>.itables table td {
    text-overflow: ellipsis;
    overflow: hidden;
}

.itables table th {
    text-overflow: ellipsis;
    overflow: hidden;
}

.itables thead input {
    width: 100%;
    padding: 3px;
    box-sizing: border-box;
}

.itables tfoot input {
    width: 100%;
    padding: 3px;
    box-sizing: border-box;
}
</style>
<div class="itables">
<table id="ac9ebab2-dce3-4bca-af37-4b8f095de9c5" class="display nowrap"style="table-layout:auto;width:auto;margin:auto;caption-side:bottom"><thead>
    <tr style="text-align: right;">

      <th>title</th>
      <th>channelTitle</th>
      <th>duration</th>
      <th>viewCount</th>
      <th>likeCount</th>
      <th>favoriteCount</th>
      <th>commentCount</th>
      <th>regionRestriction</th>
      <th>video_link</th>
    </tr>
  </thead><tbody><tr><td>Loading... (need <a href=https://mwouts.github.io/itables/troubleshooting.html>help</a>?)</td></tr></tbody></table>
<link rel="stylesheet" type="text/css" href="https://cdn.datatables.net/1.13.1/css/jquery.dataTables.min.css">
<script type="module">
    // Import jquery and DataTable
    import 'https://code.jquery.com/jquery-3.6.0.min.js';
    import dt from 'https://cdn.datatables.net/1.12.1/js/jquery.dataTables.mjs';
    dt($);

    // Define the table data
    const data = [["Create Your Own Apps Today | GeeksforGeeks", "GeeksforGeeks", "PT38S", "34923", "89", "0", "1", "NaN", "https://www.youtube.com/watch?v=LycNCWC3g18"], ["Can you solve this puzzle? | Give your answers in comments \ud83d\udc47\ud83c\udffb", "GeeksforGeeks", "PT33S", "2411", "119", "0", "11", "NaN", "https://www.youtube.com/watch?v=CGZkoNtQl_U"], ["First Ever KBC at our Offline Classes | GeeksforGeeks", "GeeksforGeeks", "PT37S", "594", "19", "0", "0", "NaN", "https://www.youtube.com/watch?v=chYg2EAKW2A"], ["Caution : 100% Relatable", "GeeksforGeeks", "PT25S", "5863", "277", "0", "2", "NaN", "https://www.youtube.com/watch?v=0NoD0Y_mAy4"], ["\ud83d\ude21\ud83d\ude24", "GeeksforGeeks", "PT11S", "4155", "164", "0", "2", "NaN", "https://www.youtube.com/watch?v=24G2og7huNY"]];

    // Define the dt_args
    let dt_args = {"order": [], "dom": "t"};
    dt_args["data"] = data;

    $(document).ready(function () {

        $('#ac9ebab2-dce3-4bca-af37-4b8f095de9c5').DataTable(dt_args);
    });
</script>
</div>



### Most viewed topics in the past 6 months.


```python
# Convert viewCount from string to integer
df['viewCount'] = df['viewCount'].astype('int')
# Sort topics by number of views
most_viewed = df.sort_values(by=['viewCount'],ascending=False)
print("Most viewed topics in the past 6 months ->")
show(most_viewed[['title', 'viewCount', 'video_link']].head(5))
# tabulate(most_viewed[['title', 'viewCount', 'video_link']].head(5), headers = 'keys', tablefmt = 'html')
```

    Most viewed topics in the past 6 months ->



<style>.itables table td {
    text-overflow: ellipsis;
    overflow: hidden;
}

.itables table th {
    text-overflow: ellipsis;
    overflow: hidden;
}

.itables thead input {
    width: 100%;
    padding: 3px;
    box-sizing: border-box;
}

.itables tfoot input {
    width: 100%;
    padding: 3px;
    box-sizing: border-box;
}
</style>
<div class="itables">
<table id="2ec3faaf-6a8a-4280-8995-943466af3e46" class="display nowrap"style="table-layout:auto;width:auto;margin:auto;caption-side:bottom"><thead>
    <tr style="text-align: right;">
      <th></th>
      <th>title</th>
      <th>viewCount</th>
      <th>video_link</th>
    </tr>
  </thead><tbody><tr><td>Loading... (need <a href=https://mwouts.github.io/itables/troubleshooting.html>help</a>?)</td></tr></tbody></table>
<link rel="stylesheet" type="text/css" href="https://cdn.datatables.net/1.13.1/css/jquery.dataTables.min.css">
<script type="module">
    // Import jquery and DataTable
    import 'https://code.jquery.com/jquery-3.6.0.min.js';
    import dt from 'https://cdn.datatables.net/1.12.1/js/jquery.dataTables.mjs';
    dt($);

    // Define the table data
    const data = [[28, "Learn System Design with GeeksforGeeks", 118339, "https://www.youtube.com/watch?v=XQEZ07JhVuA"], [20, "GeeksforGeeks Classroom Program | Now in Noida and Gurugram!", 111142, "https://www.youtube.com/watch?v=16D2cuRy5JY"], [42, "Free Summer Offline Classes on Python Programing | For Students Aged 14-21 | GeeksforGeeks", 106936, "https://www.youtube.com/watch?v=OOLXHwZzHfM"], [39, "Full Stack Development | LIVE Classes | GeeksforGeeks", 91757, "https://www.youtube.com/watch?v=cBfC9HLR9Qk"], [35, "Job Fair for Students | Till 25th May Only | GeeksforGeeks", 82854, "https://www.youtube.com/watch?v=1GEegOb3fHE"]];

    // Define the dt_args
    let dt_args = {"order": [], "dom": "t"};
    dt_args["data"] = data;

    $(document).ready(function () {

        $('#2ec3faaf-6a8a-4280-8995-943466af3e46').DataTable(dt_args);
    });
</script>
</div>




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
show(most_duration[['title', 'duration', 'duration_in_seconds', 'video_link']].head(5))
```

    Longest duration topics in the past 6 months ->



<style>.itables table td {
    text-overflow: ellipsis;
    overflow: hidden;
}

.itables table th {
    text-overflow: ellipsis;
    overflow: hidden;
}

.itables thead input {
    width: 100%;
    padding: 3px;
    box-sizing: border-box;
}

.itables tfoot input {
    width: 100%;
    padding: 3px;
    box-sizing: border-box;
}
</style>
<div class="itables">
<table id="a9bbbd28-57c9-48d4-b32e-4bbb2a659e1a" class="display nowrap"style="table-layout:auto;width:auto;margin:auto;caption-side:bottom"><thead>
    <tr style="text-align: right;">
      <th></th>
      <th>title</th>
      <th>duration</th>
      <th>duration_in_seconds</th>
      <th>video_link</th>
    </tr>
  </thead><tbody><tr><td>Loading... (need <a href=https://mwouts.github.io/itables/troubleshooting.html>help</a>?)</td></tr></tbody></table>
<link rel="stylesheet" type="text/css" href="https://cdn.datatables.net/1.13.1/css/jquery.dataTables.min.css">
<script type="module">
    // Import jquery and DataTable
    import 'https://code.jquery.com/jquery-3.6.0.min.js';
    import dt from 'https://cdn.datatables.net/1.12.1/js/jquery.dataTables.mjs';
    dt($);

    // Define the table data
    const data = [[138, "SDE Preparation in 3 hours", "PT2H55M27S", 10527, "https://www.youtube.com/watch?v=ftDoBLp-OfU"], [98, "CodeCamp Day 2 | Exploring Arrays and Problem Solving", "PT2H29M20S", 8960, "https://www.youtube.com/watch?v=OMHeYpQCCPE"], [187, "CodeCamp Day 13 | Discovering Graph Traversal Algorithms", "PT2H14M24S", 8064, "https://www.youtube.com/watch?v=hgkjJD5hb5g"], [197, "CodeCamp Day 19 | Exploring Advanced Topics in DSA", "PT2H13M39S", 8019, "https://www.youtube.com/watch?v=kLZGFHK_bIc"], [134, "CodeCamp Day 11 | Journey into Binary Trees", "PT2H10M41S", 7841, "https://www.youtube.com/watch?v=U1UKjsA4jNg"]];

    // Define the dt_args
    let dt_args = {"order": [], "dom": "t"};
    dt_args["data"] = data;

    $(document).ready(function () {

        $('#a9bbbd28-57c9-48d4-b32e-4bbb2a659e1a').DataTable(dt_args);
    });
</script>
</div>



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

fig.show()
```

```html
<html>
<head><meta charset="utf-8" /></head>
<body>
    <div>            <script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-AMS-MML_SVG"></script><script type="text/javascript">if (window.MathJax && window.MathJax.Hub && window.MathJax.Hub.Config) {window.MathJax.Hub.Config({SVG: {font: "STIX-Web"}});}</script>                <script type="text/javascript">window.PlotlyConfig = {MathJaxConfig: 'local'};</script>
        <script src="https://cdn.plot.ly/plotly-2.18.2.min.js"></script>                <div id="9c4b700c-653c-4609-9f92-d223358b5b14" class="plotly-graph-div" style="height:525px; width:100%;"></div>            <script type="text/javascript">                                    window.PLOTLYENV=window.PLOTLYENV || {};                                    if (document.getElementById("9c4b700c-653c-4609-9f92-d223358b5b14")) {                    Plotly.newPlot(                        "9c4b700c-653c-4609-9f92-d223358b5b14",                        [{"customdata":[["Create Your Own Apps Today | GeeksforGeeks"],["Can you solve this puzzle? | Give your answers in comments \ud83d\udc47\ud83c\udffb"],["First Ever KBC at our Offline Classes | GeeksforGeeks"],["Caution : 100% Relatable"],["\ud83d\ude21\ud83d\ude24"],["Hacking 101\ufffc"],["It'll work. Trust me I'm A Coder."],["DSA Offline Classes | Link In BIO"],["GeeksforGeeks Offline Classroom Program | Now Open In Noida & Gurgaon"],["Master Java Backend Development Live: Build Powerful Applications with Expert Guidance"],["Roadmap To Master Recursion? Roadmap To Master Recursion! | GeeksforGeeks"],["Coders Tell Us: \u201cPyaar Kya Hai?\u201d"],["React VS Angular VS Vue | GeeksforGeeks"],["Flutter: \"Pros and Cons\" | GeeksforGeeks"],["Ernie vs ChatGPT | GeeksforGeeks"],["What Is Web Scraping & What It Is Used For | How To Use Web Scraping | GeeksforGeeks"],["UI/UX Trends every Startup must know! | @thegeekmonk1707 |GeeksforGeeks"],["Python Libraries | Data Science | GeeksforGeeks"],["What is Queue (Updated) | Queues Explained | GeeksforGeeks"],["Solving For India Hackathon | @googlecloud X AMD | GeeksforGeeks"],["GeeksforGeeks Classroom Program | Now in Noida and Gurugram!"],["What is Arrays | Arrays Explained | New Video - GeeksforGeeks"],["Get Started with CP | Competitive Programming Tips | GeeksforGeeks"],["CAREER OPTIONS with Less CODING | GeeksforGeeks"],["Tips to learn Coding from scratch! @thegeekmonk1707  #tips #coding #learntocode #learnfromthebest"],["Tech Jobs without Degree | GeeksforGeeks"],["Best Apps to Boost your Productivity"],["DSA To Development: A Complete Coding Guide | Complete Coding Program By GeeksforGeeks"],["Learn System Design with GeeksforGeeks"],["Master DSA Today and be the Masters of Tomorrow"],["Hire Best Talent with GeeksforGeeks"],["Gate 2024: Prepare for Success! | GATE CSE 2024 | GeeksforGeeks"],["Start your writing Journey | Geek Author Badges | GeeksforGeeks"],["Google's answer to Chat GPT @thegeekmonk1707  #googlebard #chatgpt | Google bard vs Chat GPT|"],["Transformers in NLP | GeeksforGeeks"],["Job Fair for Students | Till 25th May Only | GeeksforGeeks"],["Meet POE | Chabot by QUORA ft @thegeekmonk1707"],["GeeksforGeeks Job Fair 2023"],["Link in Comments| Master Python Backend Development with Django!#python #django #djangoframework"],["Full Stack Development | LIVE Classes | GeeksforGeeks"],["Best Programming language to learn DSA | @thegeekmonk1707"],["Geek-O-Lympics 2023 | GeeksforGeeks"],["Free Summer Offline Classes on Python Programing | For Students Aged 14-21 | GeeksforGeeks"],["Side Hustle for You! @thegeekmonk1707  #sidehustle #freelancing #earnmoneyonline"],["Prepare for Gate the right way. #gate #gate2024 #gatecse #gateexam #geeksforgeeks #coding"],["Mega Job-A-Thon | Hire Top Talent for Your Company | GeeksforGeeks"],["How to Use GPT 4 | How it's better from GPT 3 | GeeksforGeeks"],["Competitive Programming Tips and Tricks | GeeksforGeeks #shorts"],["Which one are you?"],["3 Quick Tips!"],["Prompt Engineering | @thegeekmonk1707 | GeeksforGeeks"],["60 Seconds TRICK to Optimize your PYTHON Code | GeeksforGeeks #shorts"],["Kickstart your DevOps Career | GeeksforGeeks"],["Artificial Intelligence Revolution | GeeksforGeeks #shorts"],["Upgrade with skills that really matters | GeeksforGeeks"],["How to become a CTO in FUTURE | Shikhar Goel | GeeksforGeeks"],["5 Resume Mistakes to Avoid | GeeksforGeeks"],["A.I. Apps That Can Change Your Lives | GeeksforGeeks"],["Mega Job-a-thon for Working Professionals | 21st July | GeeksforGeeks"],["World's First Virtual Game | @thegeekmonk1707 | GeeksforGeeks"],["An Opportunity for Recruiters to Hire Top Tech Talent | Jobs Fair 2023 | GeeksforGeeks"],["The Art of Coding under Pressure | Competitive Programming"],["Option Hai?"],["\ud83e\udee3"],["To Switch or Not To Switch?"],["Legends say they are still running..."],["How Quantum Computing Can Change Everything | GeeksforGeeks #shorts"],["How to MOTIVATE yourself in Tough times | GeeksforGeeks #shorts"],["Should You Switch for More Money?"],["Special Rewards In #100 (Link in Comments)"],["Get ready to Master Recursion | GeeksforGeeks #shorts"],["A Step-by-Step Guide to Start with Google Cloud Services & AMD Instances | GeeksforGeeks"],["Age is Just a Number: The Story of a 14-Year-Old CEO | CodeCast | GeeksforGeeks"],["Relatable or Relatable? \ud83e\udd72"],["Master System Design | GeeksforGeeks"],["\ud83d\ude2e\ud83d\ude2e\ud83d\ude2e\ud83d\ude2e"],["ArtificiaI Intelligence Changing the world | GeeksforGeeks #shorts"],["Link in Comments!"],["How to become a Full Stack Developer | GeeksforGeeks"],["CodeCast Ep. 3 | Learn all About Competitive Programming | @PriyanshAgarwal  with Mr. Sandeep Jain"],["BHAGOOOO"],["Get ticket worth 11k | Link in Comments"],["\"Yes I know how to code\""],["Coder hun, Give up nhi karta\u2026 \ud83e\udd79"],["Arrays Practice Questions (Updated) | In C++/JAVA/Python | GeeksforGeeks"],["Jobs that AI can't replace | GeeksforGeeks"],["\ud83e\udd2fTHREADS - The Real Twitter Killer?\ud83e\udd2f || GeeksforGeeks"],["*Packs bag and goes home \ud83e\udd72*"],["Important SKILLS for CAREER Growth | GeeksforGeeks #shorts"],["CodeCamp Day 7 | Advanced Sorting techniques and Problem Solving"],["Creating Instances on Google Cloud | Solving For India - Hackathon | GeeksforGeeks"],["Learning to Build Blockchain Application | Solving For India - Hackathon | GeeksforGeeks"],["Google Cloud X AMD and GeeksforGeeks Goes to Colleges With Some Fun Twist!"],["Best Practices - API Deployment and Tech Stacks | Geek-A-Thon 2023"],["Office Video Leaked \ud83d\ude30\ud83e\udd2f"],["Presenting: GeeksforGeeks Author Badges | Honoring the Writer in You!"],["OpenAI's ChatGPT & GPT 4 | Google's BARD AI | Will AI Replace You?"],["\ud83e\udd72\ud83e\udd72"],["CodeCamp Day 2 | Exploring Arrays and Problem Solving"],["CodeCamp Day 12 | Graph Theory Unveiled"],["Day 7 | Learn Kotlin | The future of Android Development | Geek-O-Lympics 2023"],["Day 16 From Good to Great with DSA and Development | Genie Ashwani | Geek-O-Lympics"],["Close call \ud83d\ude2e\u200d\ud83d\udca8"],["CodeCamp Day 14 | Unleashing the Power of Dynamic Programming"],["The Art of Dataset Creation | Geek-A-Thon 2023"],["Don't Know How To Transition Into Product Management? Here's Your Guide | GeeksforGeeks"],["Top 10 DSA Topics | Master these To Ace In Your Career | GeeksforGeeks"],["CodeCamp Day 9 |  Unraveling Backtracking Techniques"],["Day 20 Future proofing your career | Exploring GATE and Masters"],["CodeCamp Day 18 | Conquering Divide and Conquer Strategies"],["Empowering Agrotech Innovations with GCP | Solving For India - Hackathon | GeeksforGeeks"],["Geek-O-Lympics 2023 | 1st - 31st July | GeeksforGeeks"],["Meet Shikar Goel, CTO of GeeksforGeeks | Know His Journey"],["Day 25 - PRODUCT MANAGEMENT SECRETS: From Idea to Market Success | Geek-O-Lympics 2023"],["Web 3.0 Explained | GeeksforGeeks"],["GeeksforGeeks"],["Day 24: CLOUD COMPUTING: Comparing AWS, Azure and Google Cloud | Geek-O-Lympics 2023"],["Advice for Women in Tech | ft. @PriyaVajpeyi"],["Roadmap for Test Automation Engineering | Nitesh Jain"],["Building a Blockchain App with AMD Instances on Google Cloud Platform"],["Printer Input Changes | GeeksforGeeks"],["Complete Guide to Software Testing and Automation"],["Kickstarting your Content Creation Side Hustle | Steps to Success"],["Rule the Coding World with Geek Summer Carnival | GeeksforGeeks"],["Day 31 Real-world Microservices Use Cases of Netflix Amazon Uber"],["Psychology of Deciding Your Career ft. Dhairya Gangwani | GeeksforGeeks"],["CodeCamp Day 17 | Mastering Greedy Algorithms"],["The Power of System Design: Unveiling Low-Level and High-Level Design | With @ArshGoyal"],["Complete School Guide for CBSE | Free Resources For School Students | GeeksforGeeks"],["Day 9 | The Art of Data Storytelling | How Numbers speak Louder than Words | Geek-O-Lympics 2023"],["How to avoid getting Laid Off | ft. Dr. Sneha Sharma"],["From Zero to Hero: Mastering Backend Technologies | Kushal Vijay"],["Day 22 The DevOps Mindset | Transforming Organizations for Continuous Innovation"],["All Your Queries Answered | DSA to Development Program | GeeksforGeeks"],["CodeCamp Day 11 | Journey into Binary Trees"],["CodeCamp Day 3 | Mastering Linked Lists and Problem Solving"],["Get Started With Machine Learning | GeeksforGeeks"],["Day 5 | Pushing the limits | Leveling Up In  Competitve Programming | Geek-O-Lympics 2023"],["SDE Preparation in 3 hours"],["Full Stack Development: Become a Jack of All Trades | Yajas Sardana"],["Unlock Your Coding Potential | Join the DSA to Development Journey | A Complete Career Program"],["Day 15 | How to Get Software Testing Job | Nitesh Jain | Geek-O-Lympics 2023"],["Empowering women coders || GeeksforGeeks"],["Accelerating Fitness & Sports-tech Development with GCP |  Solving For India - Hackathon"],["CodeCast Ep. 2 | SDE Without A Technical Degree? | GeeksforGeeks"],["The Art of Problem Solving: Data Structures and Algorithms @FrazMohammad"],["3 Reasons Why You Should Learn GCP | GeeksforGeeks #shorts"],["GeeksforGeeks Classroom Program in NCR"],["From Zero to AI Hero | An Inspiring Journey of an AI Engineer"],["Day 3 | Programming Languages | Bridging the gap b/w education & employment | Geek O-Lympics 2023"],["Innovating Health-tech Solutions with GCP | Solving For India - Hackathon | GeeksforGeeks"],["GATE Journey of the Toppers | Puneet Kansal"],["CodeCamp Day 15 | Diving Deeper into Dynamic Programming"],["Day 23 Django vs Flask | Choosing your correct Python framework"],["Showing Off your Geek-A-Thon Project"],["Android Development: Building Apps that Change Lives | Saumya Singh"],["Day-29 AI Powered ChatBot with ChatGPT | Geek-O-Lympics"],["Building your Fintech on Google Cloud | Solving For India - Hackathon | GeeksforGeeks"],["10 TIPS and TRICKS to Crack Internships and Placements | GeeksforGeeks"],["\ud83c\udf8aCelebrating Bi-Wizard's First Anniversary\ud83c\udf8a | Join the Best Young Coders in the World"],["Queue Practice Questions (Updated) | DSA Concepts | GeeksforGeeks"],["Software Testing | A booming Career Option | Nitesh Jain"],["ITR Filing Made Simple | Documents require to file ITR | GeeksforGeeks"],["Day 12 | Big Data Unleashed | Wondering how you stay glued to Netflix?"],["Day 26 | Understanding and Implementing Binary Trees | Anvita Bansal | Geek-O-lympics"],["Day 2 | Inside the world of Java Backend Development | Geek O-Lympics 2023"],["CodeCamp Day 1 | DSA Fundamentals and Problem-Solving Challenges"],["CodeCamp Day 6 | Sorting Techniques Demystified"],["GATE: Unlocking Opportunities in Tech | Ribhu Mukherjee"],["From Zero to AI Hero | An Inspiring Journey of an AI Engineer"],["Explore GeeksforGeeks Hiring Solutions | For Top Tech Companies | Innovative Hiring Solutions"],["Unraveling the Geek-a-thon: The What, How, and Why | Geek-A-Thon | GeeksforGeeks"],["Your First Step into \ud83e\udd47 Competitive Programming \ud83e\udd47 | No DSA Prerequisite | GeeksforGeeks"],["Roadmap to Amazon | Twinkle Bajaj - SDE at Amazon"],["Day 11 | From data to Insights | The journey of a Data Scientist | Geek-O-Lympics 2023"],["Day 1 | Cracking the Coding Interview | Santosh Mishra | Geek-O-Lympics 2023"],["CodeCamp Day 10 | Tackling Complex Problems with Recursion and Backtracking"],["Day-17 Leveraging Online Resources | Adapting to change in evolving Tech Era | Vasu Sehgal"],["Day 10 | Machine Learning 101 | Geek-O-Lympics 2023"],["Unleash Your Inner Geek: A Journey to Tech Mastery | Siddhartha Hazra"],["CodeCamp Day 4 | Understanding Stacks and Queues and Problem Solving"],["Day 13 | Chatbots and ChatGPT | Revolutionizing Human Machine Conversations | Geek-O-Lympics 2023"],["CodeCamp Day 5 | Mastering Searching Algorithms"],["Day- 19 From Idea to Startup | Navigating Entrepreneurial Journey"],["Day 4 | MERN Stack Best Practices | From Frontend to Backend | Geek-O-Lympics 2023"],["Day 8 | Power of Python in Data Science | Ashish Jangra | Geek-O-Lympics 2023"],["Day 30 Creating an Impactful and Targeted Resume | Geek-O-Lympics"],["CodeCamp Day 13 | Discovering Graph Traversal Algorithms"],["Day 6 | Demystifying System Design interviews | Conception to Reality | Geek-O-Lympics 2023"],["Skill Up This Summer with GeeksforGeeks | Free Guidance Session"],["CodeCamp Day 8 | Exploring the Magic of Recursion"],["Discovering the World of Data Science and AI | Shambhavi Gupta"],["CodeCamp Day 20 | Reviewing Key Concepts and Techniques"],["Day-18 Career Path in Tech | Navigating Job Market | Priya Vajpeyi"],["Day 14 | Data Structures for Modern Applications | DSA in era of Big Data & AI | Geek-O-Lympics 2023"],["Data Analysis Masterclass | Geek-A-Thon | GeeksforGeeks"],["CodeCamp Day 16 | Solving more and Complex Dynamic Programming Problems"],["CodeCamp Day 19 | Exploring Advanced Topics in DSA"],["CodeCamp Day 21 | Final Challenge Integrating DSA Concepts"],["Day 21 Empowering the future | Coding and Problem Solving in School"],["Day 27: Advancement in NLP( Natural Langauge Processing) | Geek-O-Lympics 2023"],["Do. Not. Touch. It."]],"hovertemplate":"duration_in_millisec=%{x}<br>viewCount=%{y}<br>title=%{customdata[0]}<extra></extra>","legendgroup":"","marker":{"color":"royalblue","symbol":"circle-open","size":7},"mode":"markers","name":"","orientation":"v","showlegend":false,"x":[38000,33000,37000,25000,11000,13000,12000,17000,28000,29000,112000,60000,95000,524000,92000,167000,123000,86000,530000,47000,38000,588000,99000,57000,60000,123000,61000,43000,36000,25000,54000,52000,72000,58000,1725000,39000,58000,31000,39000,35000,48000,46000,37000,60000,59000,56000,222000,58000,38000,32000,61000,45000,30000,46000,24000,60000,51000,210000,36000,60000,26000,33000,13000,12000,43000,15000,44000,30000,50000,17000,28000,3999000,1005000,7000,46000,4000,47000,27000,717000,1916000,7000,24000,8000,7000,648000,197000,56000,8000,35000,6099000,2408000,3384000,65000,3192000,95000,56000,2145000,5000,8960000,6600000,3040000,3186000,12000,6309000,3205000,4633000,338000,6747000,2849000,7770000,2800000,10000,2242000,2984000,1028000,50000,4466000,715000,3462000,2475000,36000,3016000,2751000,42000,4113000,1168000,7821000,3543000,56000,3569000,1897000,3370000,2802000,227000,7841000,6753000,930000,4199000,10527000,3891000,2356000,2476000,1723000,4276000,415000,4005000,56000,37000,2482000,2999000,4125000,3856000,5216000,2673000,1650000,3687000,1805000,4912000,593000,31000,734000,3313000,340000,3685000,3142000,3713000,7717000,5627000,3769000,2574000,171000,2744000,4442000,1110000,3515000,3969000,7703000,3265000,4131000,2501000,7501000,3697000,5364000,4881000,3116000,3326000,3057000,8064000,3660000,3619000,5986000,2799000,7244000,3621000,3050000,5361000,2900000,8019000,6422000,1607000,3551000,13000],"xaxis":"x","y":[34923,2411,594,5863,4155,6394,5697,30945,8680,41732,2421,5640,3054,2681,1025,1089,1037,1295,963,2442,111142,2579,2306,5810,2386,1380,2600,49337,118339,74365,14240,27927,1271,1887,2215,82854,2743,25807,30947,91757,3120,3381,106936,2802,1232,1109,3979,1677,3193,4136,1835,923,14418,935,9418,1632,1223,2856,1028,2579,1795,1910,6235,4324,4392,6477,1345,1105,2674,3711,977,12274,14244,8639,7097,7386,1000,2660,4725,5859,6344,2953,5724,5380,2704,3127,883,3204,908,1845,6133,3420,1482,2150,5286,1106,1936,5340,7043,1231,1805,1351,6614,1178,689,2943,2444,1592,761,1144,1741,1190,1656,540,1546,1090,761,1527,1030,2412,723,1058,769,25491,935,847,1527,1556,498,716,1102,2826,500,502,1412,3569,1690,1643,3183,3687,5306,1032,948,1732,1901,10438,1253,2405,2414,1230,2545,871,1354,830,737,4773,758,2284,662,613,1042,793,935,801,829,2235,24491,1867,1269,1247,595,437,3397,1746,1393,4798,1303,1014,1789,1068,2329,766,2051,681,2224,1455,1045,1025,1886,1331,1877,1753,901,949,1319,585,980,1528,1406,860,704,5007],"yaxis":"y","type":"scatter"},{"hovertemplate":"<b>Rolling mean trendline</b><br><br>duration_in_millisec=%{x}<br>viewCount=%{y} <b>(trend)</b><extra></extra>","legendgroup":"","line":{"color":"mediumpurple"},"marker":{"color":"royalblue","symbol":"circle-open","size":7},"mode":"lines","name":"","showlegend":false,"x":[4000,5000,7000,7000,7000,8000,8000,10000,11000,12000,12000,12000,13000,13000,13000,15000,17000,17000,24000,24000,25000,25000,26000,27000,28000,28000,29000,30000,30000,31000,31000,32000,33000,33000,35000,35000,36000,36000,36000,37000,37000,37000,38000,38000,38000,39000,39000,42000,43000,43000,44000,45000,46000,46000,46000,47000,47000,48000,50000,50000,51000,52000,54000,56000,56000,56000,56000,56000,57000,58000,58000,58000,59000,60000,60000,60000,60000,60000,61000,61000,65000,72000,86000,92000,95000,95000,99000,112000,123000,123000,167000,171000,197000,210000,222000,227000,338000,340000,415000,524000,530000,588000,593000,648000,715000,717000,734000,930000,1005000,1028000,1110000,1168000,1607000,1650000,1723000,1725000,1805000,1897000,1916000,2145000,2242000,2356000,2408000,2475000,2476000,2482000,2501000,2574000,2673000,2744000,2751000,2799000,2800000,2802000,2849000,2900000,2984000,2999000,3016000,3040000,3050000,3057000,3116000,3142000,3186000,3192000,3205000,3265000,3313000,3326000,3370000,3384000,3462000,3515000,3543000,3551000,3569000,3619000,3621000,3660000,3685000,3687000,3697000,3713000,3769000,3856000,3891000,3969000,3999000,4005000,4113000,4125000,4131000,4199000,4276000,4442000,4466000,4633000,4881000,4912000,5216000,5361000,5364000,5627000,5986000,6099000,6309000,6422000,6600000,6747000,6753000,7244000,7501000,7703000,7717000,7770000,7821000,7841000,8019000,8064000,8960000,10527000],"xaxis":"x","y":[null,null,null,null,6642.289705780602,6459.450183607486,5994.501585291579,4841.798985393696,3793.0227088967467,3463.6812068604117,3624.841001775619,4462.412962281543,5442.7613486356195,5732.321426728011,5988.870934499625,6056.2625439975545,5671.4016433051165,9351.459523175496,11645.454984643939,12129.265292682996,11347.204381564005,20188.804937920522,20335.797586424986,21248.47827028968,19755.910072712017,14479.786281120232,9625.331992832918,14032.928833623926,15397.710791356214,17211.13329537869,15657.64594535292,9697.376816235354,7332.770823937214,5936.539386461719,16030.902366438078,22139.53960207005,24168.781062033235,21192.08959877181,32644.38913465984,27176.416340428725,30710.336815121133,43449.99625361633,42935.54830492959,33412.95886375918,49309.130153744634,63080.25640782244,59271.87709404416,61560.244390074135,49430.194904554875,34402.603330556274,22627.531160523296,17702.315169138426,13239.566427252417,10142.675480009355,3086.472205213747,2924.940257426184,2661.329358602391,1991.7390811993193,2078.1150915786243,2237.903436124395,2164.9390362849545,5872.228284883743,9323.468639579936,10751.41828273151,10233.285802689239,7940.514399967402,2939.2887010438076,974.7381278476302,1763.8784217148525,2422.21193105684,2739.1588740536495,2806.7988563403965,2522.3756962608636,1901.70296950219,2373.2436595693894,2660.822883184449,2924.1970941897275,3040.044242304271,2894.883738945128,2375.303953574291,2285.971005779962,1953.843059687697,1655.4502370501505,1371.617892251255,1866.794742827099,2377.030212755121,2782.722065068485,2977.339621204827,2758.323743473077,2026.2524872195806,1621.5316566549684,1278.384719494452,1346.7699370290031,1730.7127730014167,2324.350443782673,2461.6016663341684,2594.586193722081,2129.0894849364076,1829.6750799466004,1684.2491925695508,1799.512207140824,1845.077465582062,1799.7417240839072,1829.8451985364525,1718.0487109855262,2278.9743667071416,2291.722323576569,2427.6260531964454,4089.6398252064555,4772.0307601070035,4723.6066036058755,4322.688654992348,3272.668116561546,1139.1235873639032,978.334530081855,1053.2159934159174,1139.8779860565746,1215.6567587820578,1964.250414285947,2377.843060642285,2514.8446387147974,3130.829588901555,3851.2684615835715,3724.6753045365244,3663.7163264401115,3372.8811139508784,2427.959651124385,1630.580516396802,1364.5314302426875,1169.2140722711695,862.3112295846503,921.9843394564756,1071.141150467551,1140.3528386501514,1170.7372786489996,1097.4346524939976,867.4067458985974,798.7386831119734,903.3553613954972,1088.3252750478741,1222.6875805915042,1327.923375327373,1464.2204834043782,1450.2786167721713,1381.8903672021584,1488.6148379166789,1444.6210282539173,1274.0219852130951,1201.999918160764,1133.31849275317,1282.9988427803858,1844.6484540275321,2069.8249609086133,2149.368751168183,1994.7624390472413,1552.635455590456,1122.5582342912855,1096.3202728805873,1013.3696927467194,1098.8464499733902,1183.5179249504934,1792.9949442659097,1940.0804448391782,2175.7831965458736,2062.831333880664,1873.8152204307792,1686.9842666678114,2390.712164889304,4249.342740559321,6471.1994898429775,7170.113692147107,6673.777861038637,5259.378590417966,3086.0352805896027,1785.4238000791768,2100.0907509708036,1942.265084142404,2106.2087178693105,1965.6813692520377,1925.881108079286,1653.2258727240117,1563.508088606462,1393.375708039036,1535.3258138649671,1552.1612379110543,1711.2174475329728,1787.6363649764842,1640.551144273221,1491.2047558149743,1411.8911729085687,1698.2989284979353,1816.7800942929075,1993.0125848943892,1976.310720961539,5352.684369099506,6589.129940501605,7287.292718808598,6473.39826775051,4917.976927246246,1363.3780499372915,2231.395098186542,2859.661059772728],"yaxis":"y","type":"scatter"}],                        {"template":{"data":{"histogram2dcontour":[{"type":"histogram2dcontour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"choropleth":[{"type":"choropleth","colorbar":{"outlinewidth":0,"ticks":""}}],"histogram2d":[{"type":"histogram2d","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmap":[{"type":"heatmap","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmapgl":[{"type":"heatmapgl","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"contourcarpet":[{"type":"contourcarpet","colorbar":{"outlinewidth":0,"ticks":""}}],"contour":[{"type":"contour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"surface":[{"type":"surface","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"mesh3d":[{"type":"mesh3d","colorbar":{"outlinewidth":0,"ticks":""}}],"scatter":[{"fillpattern":{"fillmode":"overlay","size":10,"solidity":0.2},"type":"scatter"}],"parcoords":[{"type":"parcoords","line":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolargl":[{"type":"scatterpolargl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"bar":[{"error_x":{"color":"#2a3f5f"},"error_y":{"color":"#2a3f5f"},"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"bar"}],"scattergeo":[{"type":"scattergeo","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolar":[{"type":"scatterpolar","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"histogram":[{"marker":{"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"histogram"}],"scattergl":[{"type":"scattergl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatter3d":[{"type":"scatter3d","line":{"colorbar":{"outlinewidth":0,"ticks":""}},"marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattermapbox":[{"type":"scattermapbox","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterternary":[{"type":"scatterternary","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattercarpet":[{"type":"scattercarpet","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"carpet":[{"aaxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"baxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"type":"carpet"}],"table":[{"cells":{"fill":{"color":"#EBF0F8"},"line":{"color":"white"}},"header":{"fill":{"color":"#C8D4E3"},"line":{"color":"white"}},"type":"table"}],"barpolar":[{"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"barpolar"}],"pie":[{"automargin":true,"type":"pie"}]},"layout":{"autotypenumbers":"strict","colorway":["#636efa","#EF553B","#00cc96","#ab63fa","#FFA15A","#19d3f3","#FF6692","#B6E880","#FF97FF","#FECB52"],"font":{"color":"#2a3f5f"},"hovermode":"closest","hoverlabel":{"align":"left"},"paper_bgcolor":"white","plot_bgcolor":"#E5ECF6","polar":{"bgcolor":"#E5ECF6","angularaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"radialaxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"ternary":{"bgcolor":"#E5ECF6","aaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"baxis":{"gridcolor":"white","linecolor":"white","ticks":""},"caxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"coloraxis":{"colorbar":{"outlinewidth":0,"ticks":""}},"colorscale":{"sequential":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"sequentialminus":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"diverging":[[0,"#8e0152"],[0.1,"#c51b7d"],[0.2,"#de77ae"],[0.3,"#f1b6da"],[0.4,"#fde0ef"],[0.5,"#f7f7f7"],[0.6,"#e6f5d0"],[0.7,"#b8e186"],[0.8,"#7fbc41"],[0.9,"#4d9221"],[1,"#276419"]]},"xaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"yaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"scene":{"xaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"yaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"zaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2}},"shapedefaults":{"line":{"color":"#2a3f5f"}},"annotationdefaults":{"arrowcolor":"#2a3f5f","arrowhead":0,"arrowwidth":1},"geo":{"bgcolor":"white","landcolor":"#E5ECF6","subunitcolor":"white","showland":true,"showlakes":true,"lakecolor":"white"},"title":{"x":0.05},"mapbox":{"style":"light"}}},"xaxis":{"anchor":"y","domain":[0.0,1.0],"title":{"text":"duration_in_millisec"}},"yaxis":{"anchor":"x","domain":[0.0,1.0],"title":{"text":"viewCount"}},"legend":{"tracegroupgap":0,"font":{"family":"sans-serif","size":12,"color":"Black"},"x":0.01,"y":0.98,"traceorder":"normal","bgcolor":"LightSteelBlue","bordercolor":"dimgray","borderwidth":2},"title":{"text":"No. of Views V/S Length of Videos"},"showlegend":true},                        {"responsive": true}                    ).then(function(){

var gd = document.getElementById('9c4b700c-653c-4609-9f92-d223358b5b14');
var x = new MutationObserver(function (mutations, observer) {{
        var display = window.getComputedStyle(gd).display;
        if (!display || display === 'none') {{
            console.log([gd, 'removed!']);
            Plotly.purge(gd);
            observer.disconnect();
        }}
}});

// Listen for the removal of the full notebook cells
var notebookContainer = gd.closest('#notebook-container');
if (notebookContainer) {{
    x.observe(notebookContainer, {childList: true});
}}

// Listen for the clearing of the current output cell
var outputEl = gd.closest('.output');
if (outputEl) {{
    x.observe(outputEl, {childList: true});
}}

                        })                };                            </script>        </div>
</body>
</html>
```

From the graph above you can see that no. of views is extremely high when the duration of videos is less. As the duration increases there is steep decline in viewer count until it reaches a stable value wherein even 10million millisec duration videos get a low viewer count.


```python
!jupyter nbconvert --to markdown filename.ipynb
```