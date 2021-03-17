---
layout: post
title: Creating a /r/Wallstreetbets skimmer
---


## Introduction

Who even cares what WSB thinks about stocks? Well, probably no one. At least at the time that I started on this project. Since then, the whole GME debachle started, and honestly, WSB has been annoyingly difficult to read, cause if you're not all in for GME, BB, or AMC, then there's not much there for you right now. It's time for some new meme stocks dammit. 

So initially, I started this project to try to quantify the sentiment of WSB on any mentioned stock tickers. And now it's sorta morphed into a way to find the underlying discussion that gets clouded by all the APE DIAMOND STRONG ME LIKE STONK HANDS.


## Getting posts

First things first, we are going to use python to make this happen. I used the requests package and the [pushshift.io](https://pushshift.io/api-parameters/) api to easily grab data from reddit. 

So, I first created a DataAPI class that could be used to easily get different types of data from the pushshift api.

```python 
import requests
import datetime


class DataAPI:
    def __init__(self, interest):
        self.interest = interest
```

When we create and instance of this class, we'll pass in a filename of a .csv containing the stocks that we are interested in (stored as `self.interest`). For my purposes, I just selected tickers that were >$500 million in market cap.

Next, we have the `get_tickers` function to obtain an actual list of the tickers we care about. 
```python

    def get_tickers(self):
        with open(self.interest, "r") as file:
            lines = file.readlines()

        tickers = [x.split(",")[0] for x in lines[1:]]

        return tickers
```

Following that, we have three functions to request data from the pushshift api: `get_submissions`, `get_comments`, and `get_historical_comments`. All three of these functions start by making a request to the pushshift.io api. In each case, we pass the name of the subreddit we want to query and the number of results we want to receive. The maximum number of results that will be returned from a single request is 100. `get_historical_comments` obtains comments up to the number specified by making multiple requests and utilizing the `before` parameter of the api. 

```python

    def get_submissions(self, subreddit):
        r = requests.get(
            "https://api.pushshift.io/reddit/submission/search",
            params={"subreddit": subreddit, "limit": 500},
        )
        raw = r.json()
        titles = [x["title"] for x in raw["data"]]
        bodies = [x["selftext"] if not x["is_self"] else "Image" for x in raw["data"]]
        return titles, bodies

    def get_comments(self, subreddit):
        r = requests.get(
            "https://api.pushshift.io/reddit/comment/search",
            params={"subreddit": subreddit, "size": 100},
        )
        raw = r.json()
        comments = [x["body"] for x in raw["data"]]

        return comments

    def get_historical_comments(self, subreddit, number):
        r = requests.get(
            "https://api.pushshift.io/reddit/comment/search",
            params={"subreddit": subreddit, "size": 100},
        )
        raw = r.json()
        comments = [x["body"] for x in raw["data"]]
        before_value = raw["data"][-1]["created_utc"]
        yesterday = datetime.date.today() - datetime.timedelta(2)
        yesterday = int(
            datetime.datetime.timestamp(
                datetime.datetime.combine(yesterday, datetime.datetime.min.time())
            )
        )
        while len(comments) < number:  # or (before_value > yesterday):
            r = requests.get(
                "https://api.pushshift.io/reddit/comment/search",
                params={"subreddit": subreddit, "size": 100, "before": before_value},
            )
            raw = r.json()
            comments2 = [x["body"] for x in raw["data"]]
            for c in comments2:
                comments.append(c)
            before_value = raw["data"][-1]["created_utc"]
            print(f"{len(comments)} have been collected.")
        return comments
```

Finally, we have the `identify_tickers` function. The data from one of the requesting functions is passed here to be parsed and find mentions of stocks that are in our `self.interest` list. Comments that have at least one match are appended to a list, and the matching tickers are appended to another list. These lists are returned for further processing. 

```python
    def identify_tickers(self, data):
        tickers = self.get_tickers()
        commentsWithTicker = []
        usedTickers = []
        for comment in data:
            commentTickers = [
                i.strip("$") for i in comment.split(" ") if i.strip("$") in tickers
            ]
            if commentTickers:
                commentsWithTicker.append(comment)
                usedTickers.append(commentTickers)

        return commentsWithTicker, usedTickers
```

And that's the end of the DataAPI class. So now that we are able to access the data, we can actually do stuff with it. That will be covered in the next post.