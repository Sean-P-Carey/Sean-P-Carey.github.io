---
Title: "Data Collection - Political Twitter Part I"
date: 2020-01-23
tags: [twitter, api, tweepy, practicum]
header:
   image: "/images/background.jpg"
excerpt: "Collecting Tweets from American Politicians using Tweepy and Python"
---

## Data Collection

The use of Social Media as a platform in American politics has been growing in popularity since the 2008 election cycle. Twitter has become the platform of choice for most members of both the Legislative and the Executive Branches. The goal of this project will be to step through an  analyze the vocabulary used by each political party. This blog series will have four parts:  

* Part I: Collection of the Twitter Data
* Part II: Cleaning and Preprocessing of the text data
* Part III: Exploratory Data Analysis of the cleaned text data
* Part IV: Classify the Tweets using a Deep Learning Model  



At the end of this post we will have a labeled dataset with unprocessed Tweets from American Politicians.


```python

import pandas as pd
import numpy as np
import tweepy
import tabula
import os
import pickle
import boto3
s3 = boto3.resource('s3')
bucket_name = "msds-practicum-carey"

c_key = os.environ.get('tw_c_key')
c_sec = os.environ.get('tw_c_sec')
atk = os.environ.get('tw_ac_tok')
ats = os.environ.get('tw_ac_sec')

auth = tweepy.OAuthHandler(c_key, c_sec)
auth.set_access_token(atk, ats)

api = tweepy.API(auth, wait_on_rate_limit_notify=True,
                wait_on_rate_limit=True)

pd.set_option('display.max_rows', 200)
```

***


```python
# read in data
congress_url = "https://theunitedstates.io/congress-legislators/legislators-current.csv"
congress_df = pd.read_csv(congress_url,
                          usecols=['last_name',
                                   'first_name',
                                   'full_name',
                                   'party',
                                   'type',
                                   'state' ,
                                   'twitter'])

```

### Check for missing Twitter Handles


```python
# filter for records with missing twitter handle
 congress_df[congress_df['twitter'].isna()]

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
      <th>last_name</th>
      <th>first_name</th>
      <th>full_name</th>
      <th>type</th>
      <th>state</th>
      <th>party</th>
      <th>twitter</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>33</th>
      <td>Amash</td>
      <td>Justin</td>
      <td>Justin Amash</td>
      <td>rep</td>
      <td>MI</td>
      <td>Independent</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>62</th>
      <td>Clay</td>
      <td>Wm.</td>
      <td>Wm. Lacy Clay</td>
      <td>rep</td>
      <td>MO</td>
      <td>Democrat</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>182</th>
      <td>Peterson</td>
      <td>Collin</td>
      <td>Collin C. Peterson</td>
      <td>rep</td>
      <td>MN</td>
      <td>Democrat</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>314</th>
      <td>Kaine</td>
      <td>Timothy</td>
      <td>Tim Kaine</td>
      <td>sen</td>
      <td>VA</td>
      <td>Democrat</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>372</th>
      <td>Comer</td>
      <td>James</td>
      <td>James Comer</td>
      <td>rep</td>
      <td>KY</td>
      <td>Republican</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>425</th>
      <td>Gianforte</td>
      <td>Greg</td>
      <td>Greg Gianforte</td>
      <td>rep</td>
      <td>MT</td>
      <td>Republican</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>534</th>
      <td>Bishop</td>
      <td>Dan</td>
      <td>Dan Bishop</td>
      <td>rep</td>
      <td>NC</td>
      <td>Republican</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>535</th>
      <td>Murphy</td>
      <td>Gregory</td>
      <td>Gregory F. Murphy</td>
      <td>rep</td>
      <td>NC</td>
      <td>Republican</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>536</th>
      <td>Loeffler</td>
      <td>Kelly</td>
      <td>Kelly Loeffler</td>
      <td>sen</td>
      <td>GA</td>
      <td>Republican</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



A few twitter handles are missing. Since it is a small numbe I will manually fill these in.

Georgia swore in a new Senator to replace retired Sen. Johnny Isakson. I've elected to keep Johnny Isakson's name in the list in order to capture all of his tweets


```python
# fill in missing twitter handles
congress_df.at[33, 'twitter'] = 'justinamash'

congress_df.at[62, 'twitter'] = 'LacyClayMO1'

congress_df.at[182, 'twitter'] = "collinpeterson"

congress_df.at[314, 'twitter'] = "timkaine"

congress_df.at[372, 'twitter'] = "KYComer"

congress_df.at[425, 'twitter'] = "GregForMontana"

congress_df.at[534, 'twitter'] = "jdanbishop"

congress_df.at[535, 'twitter'] = "DrGregMurphy1"

congress_df.at[536, 'twitter'] = "SenatorLoeffler"

print(f"Missing Twitter Handles: {len(congress_df[congress_df['twitter'].isna()])}")


```


```python
# group by political party
congress_df.groupby(by='party').count()
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
      <th>last_name</th>
      <th>first_name</th>
      <th>full_name</th>
      <th>type</th>
      <th>state</th>
      <th>twitter</th>
    </tr>
    <tr>
      <th>party</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Democrat</th>
      <td>281</td>
      <td>281</td>
      <td>281</td>
      <td>281</td>
      <td>281</td>
      <td>281</td>
    </tr>
    <tr>
      <th>Independent</th>
      <td>3</td>
      <td>3</td>
      <td>3</td>
      <td>3</td>
      <td>3</td>
      <td>3</td>
    </tr>
    <tr>
      <th>Republican</th>
      <td>253</td>
      <td>253</td>
      <td>253</td>
      <td>253</td>
      <td>253</td>
      <td>253</td>
    </tr>
  </tbody>
</table>
</div>


## Independent members of Congress

Three members of this Congress are *Independent*. Which means that they do not belong to any one political party. They may not belong to a party but they do have *Conservative* or *Liberal* political leanings. We can relabel them with the party with which they most closely align. Doing this allows us to relabel them as either *Liberal* or *Conservative*.

```python
# filter by Independents
congress_df[congress_df['party']=='Independent']
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
      <th>last_name</th>
      <th>first_name</th>
      <th>full_name</th>
      <th>type</th>
      <th>state</th>
      <th>party</th>
      <th>twitter</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>8</th>
      <td>Sanders</td>
      <td>Bernard</td>
      <td>Bernard Sanders</td>
      <td>sen</td>
      <td>VT</td>
      <td>Independent</td>
      <td>SenSanders</td>
    </tr>
    <tr>
      <th>33</th>
      <td>Amash</td>
      <td>Justin</td>
      <td>Justin Amash</td>
      <td>rep</td>
      <td>MI</td>
      <td>Independent</td>
      <td>justinamash</td>
    </tr>
    <tr>
      <th>287</th>
      <td>King</td>
      <td>Angus</td>
      <td>Angus S. King, Jr.</td>
      <td>sen</td>
      <td>ME</td>
      <td>Independent</td>
      <td>SenAngusKing</td>
    </tr>
  </tbody>
</table>
</div>




```python
# relabel the Independents
congress_df.at[8, 'party'] = 'Democrat'
congress_df.at[33, 'party'] = 'Republican'
congress_df.at[287, 'party'] = 'Democrat'

# create new column of Liberal or Conservative labels.
congress_df['lean'] = np.where(congress_df['party']=='Democrat','L', 'C')
congress_df
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
      <th>last_name</th>
      <th>first_name</th>
      <th>full_name</th>
      <th>type</th>
      <th>state</th>
      <th>party</th>
      <th>twitter</th>
      <th>lean</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Brown</td>
      <td>Sherrod</td>
      <td>Sherrod Brown</td>
      <td>sen</td>
      <td>OH</td>
      <td>Democrat</td>
      <td>SenSherrodBrown</td>
      <td>L</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Cantwell</td>
      <td>Maria</td>
      <td>Maria Cantwell</td>
      <td>sen</td>
      <td>WA</td>
      <td>Democrat</td>
      <td>SenatorCantwell</td>
      <td>L</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Cardin</td>
      <td>Benjamin</td>
      <td>Benjamin L. Cardin</td>
      <td>sen</td>
      <td>MD</td>
      <td>Democrat</td>
      <td>SenatorCardin</td>
      <td>L</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Carper</td>
      <td>Thomas</td>
      <td>Thomas R. Carper</td>
      <td>sen</td>
      <td>DE</td>
      <td>Democrat</td>
      <td>SenatorCarper</td>
      <td>L</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Casey</td>
      <td>Robert</td>
      <td>Robert P. Casey, Jr.</td>
      <td>sen</td>
      <td>PA</td>
      <td>Democrat</td>
      <td>SenBobCasey</td>
      <td>L</td>
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
    </tr>
    <tr>
      <th>532</th>
      <td>Golden</td>
      <td>Jared</td>
      <td>Jared F. Golden</td>
      <td>rep</td>
      <td>ME</td>
      <td>Democrat</td>
      <td>repgolden</td>
      <td>L</td>
    </tr>
    <tr>
      <th>533</th>
      <td>Keller</td>
      <td>Fred</td>
      <td>Fred Keller</td>
      <td>rep</td>
      <td>PA</td>
      <td>Republican</td>
      <td>RepFredKeller</td>
      <td>C</td>
    </tr>
    <tr>
      <th>534</th>
      <td>Bishop</td>
      <td>Dan</td>
      <td>Dan Bishop</td>
      <td>rep</td>
      <td>NC</td>
      <td>Republican</td>
      <td>jdanbishop</td>
      <td>C</td>
    </tr>
    <tr>
      <th>535</th>
      <td>Murphy</td>
      <td>Gregory</td>
      <td>Gregory F. Murphy</td>
      <td>rep</td>
      <td>NC</td>
      <td>Republican</td>
      <td>DrGregMurphy1</td>
      <td>C</td>
    </tr>
    <tr>
      <th>536</th>
      <td>Loeffler</td>
      <td>Kelly</td>
      <td>Kelly Loeffler</td>
      <td>sen</td>
      <td>GA</td>
      <td>Republican</td>
      <td>SenatorLoeffler</td>
      <td>C</td>
    </tr>
  </tbody>
</table>
<p>537 rows × 8 columns</p>
</div>



## Use the Twitter API to collect Tweets  

Twitter provides an excellent API for programmatically accessing Tweets from anybody with a Twitter account. Since we already have the Twitter handle for each membe of congress we can use the API to pull their "tweets". One drawback of pulling tweets from an individual user is that Twitter limits us to the most recent 3200 tweets per user.


```python
def get_tweets(handle):
	# blank list
    print(f'...Getting Tweets for {handle}')
    tweets = []
    try:
    # get the most recent 200 tweets
        new_tweets = api.user_timeline(screen_name = handle,
                                       count=200)

    # add new tweets one by one to end of tweets list
        tweets.extend(new_tweets)

    # get oldest tweet from list
        old = tweets[-1].id - 1

        while len(new_tweets) > 0:


            # get next 200 tweets
            new_tweets = api.user_timeline(screen_name = handle,
                                           count=200,
                                           max_id=old)

        #add current batch to the tweets list
            tweets.extend(new_tweets)

        # update the old var to match the oldest tweet currently in
            old = tweets[-1].id - 1

        tweet_tab = [tweet.text for tweet in tweets]

    except tweepy.TweepError:
        print(f'error with {handle} in function')
        pass    


    return tweet_tab
```


```python

liberal_handle_list = congress_df[congress_df['lean']=='L'].twitter
conservative_handle_list = congress_df[congress_df['lean']=='C'].twitter
```
### Get Tweets from Democratic members of Congress  

The output of this  is a list of 728,175 tweets from Democrat Senators and Representatives

```python

lib_tweets = []
for name in liberal_handle_list:

    try:
        tweets_temp = get_tweets(name)
        lib_tweets.extend(tweets_temp)
        with open('outdata/lib_list.pkl', 'wb') as f:
            pickle.dump(lib_tweets, f)
    except:
        print(f"problem with {name} in loop")
```


### Get Tweets from Republican Members of Congress  

The output of this  is a list of 589,235 tweets from Republican Senators and Representatives.


```python
conservative_tweets = []
for name in conservative_handle_list:
    try:
        tweets_temp = get_tweets(name)
        conservative_tweets.extend(tweets_temp)
        with open('outdata/con_list.pkl', 'wb') as f:
            pickle.dump(conservative_tweets, f)
        print(f'con_tweets len: {len(conservative_tweets)}')

    except:
        print(f"problem with {name} in loop")


```



### Manually fix the problems that arose for a few Conservative Handles

Some of the twitter handles did not work with the API. It turns out that the list had wrong info for some of these members. We can manually find the correct handle and collect the data.

```python

# Rep Rob Marshal's twitter handle needed to be updated     
marsh_tweets = get_tweets('RogerMarshallMD')
conservative_tweets.extend(marsh_tweets)
with open('outdata/con_list.pkl', 'wb') as f:
            pickle.dump(conservative_tweets, f)
```

    ...Getting Tweets for RogerMarshallMD



```python
# Rep. Lance Gooden's twitter handle needed to be updated.
gooden_tweets = get_tweets('Lancegooden')
conservative_tweets.extend(gooden_tweets)
with open('outdata/con_list.pkl', 'wb') as f:
            pickle.dump(conservative_tweets, f)
```

    ...Getting Tweets for Lancegooden


## Add Current Executive Branch

The current Executive branch tweets *a lot*. They should be included in this analysis.

```python
# Add President Trump to the Conservative Tweet List
trump_tweets = get_tweets('realDonaldTrump')
conservative_tweets.extend(trump_tweets)
with open('outdata/con_list.pkl', 'wb') as f:
            pickle.dump(conservative_tweets, f)
print(f'{len(conservative_tweets)}')
```

    ...Getting Tweets for realDonaldTrump
    594399



```python
# Define a function for adding to the conservative list
def add_tweets_to_con_list(handle):
    temp_tweets = get_tweets(handle)
    conservative_tweets.extend(temp_tweets)
    with open('outdata/con_list.pkl', 'wb') as f:
            pickle.dump(conservative_tweets, f)
    print(f'Conservative Tweets: {len(conservative_tweets)}')

```


```python
# add VP Pence
add_tweets_to_con_list('VP')
```

    ...Getting Tweets for VP
    597644


### Add the previous Executive Branch and Current Democratic Presidential Candidates (that aren't in congress)

The Previous Executive branch pioneered the use of Social Media for Politics. Also, this is an election year. The democrats have a lot of folks vying for the Democratic Nomination. A few of them are current members of congress so we already have their tweets. However, it makes sense to add those who are not members of congress  to this dataset.

```python
def add_tweets_to_lib_list(handle):
    temp_tweets = get_tweets(handle)
    lib_tweets.extend(temp_tweets)
    with open('outdata/lib_list.pkl', 'wb') as f:
            pickle.dump(lib_tweets, f)
    print(f'Liberal Tweets: {len(lib_tweets)}')
```


```python
add_tweets_to_lib_list('BarackObama')
add_tweets_to_lib_list('JoeBiden')
add_tweets_to_lib_list('MikeBloomberg')
add_tweets_to_lib_list('DevalPatrick')
add_tweets_to_lib_list('TomSteyer')
add_tweets_to_lib_list('PeteButtigieg')
add_tweets_to_lib_list('JohnDelaney')
add_tweets_to_lib_list('AndrewYang')


```

    ...Getting Tweets for BarackObama
    Liberal Tweets: 731403
    ...Getting Tweets for JoeBiden
    Liberal Tweets: 734630
    ...Getting Tweets for MikeBloomberg
    Liberal Tweets: 737863
    ...Getting Tweets for DevalPatrick
    Liberal Tweets: 739782
    ...Getting Tweets for TomSteyer
    Liberal Tweets: 742994
    ...Getting Tweets for PeteButtigieg
    Liberal Tweets: 746216
    ...Getting Tweets for JohnDelaney
    Liberal Tweets: 749432
    ...Getting Tweets for AndrewYang
    Liberal Tweets: 752662



```python

```


```python
s3.meta.client.upload_file('outdata/lib_list.pkl', bucket_name, 'lib_list.pkl')
s3.meta.client.upload_file('outdata/con_list.pkl', bucket_name, 'con_list.pkl')
```

## Check for Data Imbalance  

The data does not look terribly imbalanced. Liberals represent a larger proportion of the tweets than conservatives.


```python
percent_con_tweets = (len(conservative_tweets)) / (len(lib_tweets) + len(conservative_tweets))
percent_lib_tweets = (len(lib_tweets)) / (len(lib_tweets) + len(conservative_tweets))

print(f'Proportion of Tweets from Conservatives: {percent_con_tweets}')
print(f'Proportion of Tweets from Liberals: {percent_lib_tweets}')
```

    Proportion of Tweets from Conservatives: 0.442598936833577
    Proportion of Tweets from Liberals: 0.557401063166423


## Create a Combined Dataframe of tweets with labels



```python
# read in lists of tweets

# create liberal and Conservative Dataframes    
lib_df = pd.DataFrame(columns=['tweet', 'class'])
con_df = pd.DataFrame(columns=['tweet', 'class'])

# Fill liberal dataframe
lib_df['tweet'] = lib_list
lib_df['class'] = "L"

# fill conservative dataframe
con_df['tweet'] = con_list
con_df['class'] = "C"



#combine the liberal and conservative dataframes
tweet_df = pd.concat([lib_df, con_df])

# Randomly shuffle the dataframe
tweet_df = tweet_df.sample(frac=1)

# reset the index of the complete dataframe
tweet_df.reset_index(drop=True, inplace = True)

# view dataframe
tweet_df
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
      <th>tweet</th>
      <th>class</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>RT @aafb: Congrats to ⁦@RepOHalleran⁩ &amp;amp; ⁦@...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Great to meet the new Lake County Farm Bureau ...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Congratulations to @waynestcollege women's rug...</td>
      <td>C</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Great to meet with the Erickson Air Crane team...</td>
      <td>C</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Always wonderful to be part of the Back to Sch...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>1350301</th>
      <td>We should be upholding the National Environmen...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>1350302</th>
      <td>If anything is to be investigated, I think we ...</td>
      <td>C</td>
    </tr>
    <tr>
      <th>1350303</th>
      <td>TODAY: Federal judge rules in favor of House R...</td>
      <td>C</td>
    </tr>
    <tr>
      <th>1350304</th>
      <td>In the words of an old proverb, "A hit dog wil...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>1350305</th>
      <td>The new EPA regs are pure fantasy. http://t.co...</td>
      <td>C</td>
    </tr>
  </tbody>
</table>
<p>1350306 rows × 2 columns</p>
</div>




```python
with open('outdata/tweet_df.pkl', 'wb') as f:
            pickle.dump(tweet_df, f)
```


```python
# save dataframe to pickle to AWS S3
s3.meta.client.upload_file('outdata/tweet_df.pkl',
                           bucket_name,
                           'tweet_df.pkl')

```


```python
os.remove('outdata/con_list.pkl')
os.remove('outdata/lib_list.pkl')
os.remove('outdata/tweet_df.pkl')
```
