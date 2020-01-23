---
Title: "Data Collection"
date: 2020-01-23
tags: [twitter, api, tweepy, practicum]
header:
   image: "/images/background.jpg"
excerpt: "Collecting Tweets from American Politicians using Tweepy and Python"
---

## Data Collection  

The output from this notebook will be a two column dataframe that contains raw tweets in one column and the the classification (Liberal or Conservative) in the second column.


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




```python

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
congress_df.at[8, 'party'] = 'Democrat'
congress_df.at[33, 'party'] = 'Republican'
congress_df.at[287, 'party'] = 'Democrat'


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



## Get tweets from congress


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


```python

lib_tweets = []
for name in liberal_handle_list:

    try:
        tweets_temp = get_tweets(name)
        lib_tweets.extend(tweets_temp)
        with open('outdata/lib_list.pkl', 'wb') as f:
            pickle.dump(lib_tweets, f)
        print(f'Lib_tweets len: {len(lib_tweets)}')
    except:
        print(f"problem with {name} in loop")




```

    ...Getting Tweets for SenSherrodBrown
    Lib_tweets len: 3245
    ...Getting Tweets for SenatorCantwell
    Lib_tweets len: 6458
    ...Getting Tweets for SenatorCardin
    Lib_tweets len: 9690
    ...Getting Tweets for SenatorCarper
    Lib_tweets len: 12892
    ...Getting Tweets for SenBobCasey
    Lib_tweets len: 16110
    ...Getting Tweets for SenFeinstein
    Lib_tweets len: 19312
    ...Getting Tweets for SenAmyKlobuchar
    Lib_tweets len: 19939
    ...Getting Tweets for SenatorMenendez
    Lib_tweets len: 23186
    ...Getting Tweets for SenSanders
    Lib_tweets len: 26424
    ...Getting Tweets for SenStabenow
    Lib_tweets len: 29632
    ...Getting Tweets for SenatorTester
    Lib_tweets len: 32829
    ...Getting Tweets for SenWhitehouse
    Lib_tweets len: 36057
    ...Getting Tweets for SenatorDurbin
    Lib_tweets len: 39303
    ...Getting Tweets for SenJeffMerkley
    Lib_tweets len: 42504
    ...Getting Tweets for SenJackReed
    Lib_tweets len: 45751
    ...Getting Tweets for SenatorShaheen
    Lib_tweets len: 48984
    ...Getting Tweets for SenatorTomUdall
    Lib_tweets len: 52189
    ...Getting Tweets for MarkWarner
    Lib_tweets len: 55399
    ...Getting Tweets for GillibrandNY
    Lib_tweets len: 56090
    ...Getting Tweets for ChrisCoons
    Lib_tweets len: 59300
    ...Getting Tweets for Sen_JoeManchin
    Lib_tweets len: 62506
    ...Getting Tweets for SenatorBaldwin
    Lib_tweets len: 65751
    ...Getting Tweets for RepKarenBass
    Lib_tweets len: 68953
    ...Getting Tweets for SenatorBennet
    Lib_tweets len: 72183
    ...Getting Tweets for SanfordBishop
    Lib_tweets len: 74253
    ...Getting Tweets for BlumenauerMedia
    Lib_tweets len: 74605
    ...Getting Tweets for SenBlumenthal
    Lib_tweets len: 77840
    ...Getting Tweets for GKButterfield
    Lib_tweets len: 81056
    ...Getting Tweets for RepAndreCarson
    Lib_tweets len: 84291
    ...Getting Tweets for USRepKCastor
    Lib_tweets len: 87499
    ...Getting Tweets for RepJudyChu
    Lib_tweets len: 90728
    ...Getting Tweets for RepCicilline
    Lib_tweets len: 93962
    ...Getting Tweets for RepYvetteClarke
    Lib_tweets len: 97185
    ...Getting Tweets for LacyClayMO1
    Lib_tweets len: 100407
    ...Getting Tweets for RepCleaver
    Lib_tweets len: 103609
    ...Getting Tweets for WhipClyburn
    Lib_tweets len: 106845
    ...Getting Tweets for RepCohen
    Lib_tweets len: 110049
    ...Getting Tweets for GerryConnolly
    Lib_tweets len: 113276
    ...Getting Tweets for RepJimCooper
    Lib_tweets len: 116514
    ...Getting Tweets for RepJimCosta
    Lib_tweets len: 119138
    ...Getting Tweets for RepJoeCourtney
    Lib_tweets len: 122366
    ...Getting Tweets for RepCuellar
    Lib_tweets len: 125585
    ...Getting Tweets for RepDannyDavis
    Lib_tweets len: 127974
    ...Getting Tweets for RepSusanDavis
    Lib_tweets len: 130348
    ...Getting Tweets for RepPeterDeFazio
    Lib_tweets len: 131906
    ...Getting Tweets for RepDianaDeGette
    Lib_tweets len: 135135
    ...Getting Tweets for RosaDeLauro
    Lib_tweets len: 138363
    ...Getting Tweets for RepTedDeutch
    Lib_tweets len: 141608
    ...Getting Tweets for RepLloydDoggett
    Lib_tweets len: 144826
    ...Getting Tweets for USRepMikeDoyle
    Lib_tweets len: 146920
    ...Getting Tweets for RepEliotEngel
    Lib_tweets len: 150166
    ...Getting Tweets for RepAnnaEshoo
    Lib_tweets len: 152749
    ...Getting Tweets for RepMarciaFudge
    Lib_tweets len: 155960
    ...Getting Tweets for RepGaramendi
    Lib_tweets len: 159204
    ...Getting Tweets for RepAlGreen
    Lib_tweets len: 161379
    ...Getting Tweets for RepraulGrijalva
    Lib_tweets len: 164604
    ...Getting Tweets for RepHastingsFL
    Lib_tweets len: 166634
    ...Getting Tweets for MartinHeinrich
    Lib_tweets len: 169840
    ...Getting Tweets for RepBrianHiggins
    Lib_tweets len: 173055
    ...Getting Tweets for JAHimes
    Lib_tweets len: 176276
    ...Getting Tweets for MazieHirono
    Lib_tweets len: 179524
    ...Getting Tweets for LeaderHoyer
    Lib_tweets len: 182753
    ...Getting Tweets for JacksonLeeTX18
    Lib_tweets len: 185972
    ...Getting Tweets for RepEBJ
    Lib_tweets len: 189193
    ...Getting Tweets for RepHankJohnson
    Lib_tweets len: 192434
    ...Getting Tweets for RepMarcyKaptur
    Lib_tweets len: 195647
    ...Getting Tweets for USRepKeating
    Lib_tweets len: 197148
    ...Getting Tweets for RepRonKind
    Lib_tweets len: 198738
    ...Getting Tweets for JimLangevin
    Lib_tweets len: 201985
    ...Getting Tweets for RepRickLarsen
    Lib_tweets len: 205196
    ...Getting Tweets for RepJohnLarson
    Lib_tweets len: 208417
    ...Getting Tweets for SenatorLeahy
    Lib_tweets len: 211639
    ...Getting Tweets for RepBarbaraLee
    Lib_tweets len: 214865
    ...Getting Tweets for RepJohnLewis
    Lib_tweets len: 216852
    ...Getting Tweets for RepLipinski
    Lib_tweets len: 220074
    ...Getting Tweets for DaveLoebsack
    Lib_tweets len: 221404
    ...Getting Tweets for RepZoeLofgren
    Lib_tweets len: 223890
    ...Getting Tweets for NitaLowey
    Lib_tweets len: 227095
    ...Getting Tweets for RepBenRayLujan
    Lib_tweets len: 230317
    ...Getting Tweets for RepStephenLynch
    Lib_tweets len: 233539
    ...Getting Tweets for RepMaloney
    Lib_tweets len: 236787
    ...Getting Tweets for SenMarkey
    Lib_tweets len: 240022
    ...Getting Tweets for DorisMatsui
    Lib_tweets len: 243259
    ...Getting Tweets for BettyMcCollum04
    Lib_tweets len: 246473
    ...Getting Tweets for RepMcGovern
    Lib_tweets len: 249677
    ...Getting Tweets for RepMcNerney
    Lib_tweets len: 252103
    ...Getting Tweets for RepGregoryMeeks
    Lib_tweets len: 255348
    ...Getting Tweets for RepGwenMoore
    Lib_tweets len: 258555
    ...Getting Tweets for senmurphyoffice
    Lib_tweets len: 259763
    ...Getting Tweets for PattyMurray
    Lib_tweets len: 262973
    ...Getting Tweets for RepJerryNadler
    Lib_tweets len: 266181
    ...Getting Tweets for GraceNapolitano
    Lib_tweets len: 269138
    ...Getting Tweets for RepRichardNeal
    Lib_tweets len: 271246
    ...Getting Tweets for EleanorNorton
    Lib_tweets len: 274467
    ...Getting Tweets for FrankPallone
    Lib_tweets len: 277689
    ...Getting Tweets for BillPascrell
    Lib_tweets len: 280934
    ...Getting Tweets for SpeakerPelosi
    Lib_tweets len: 284166
    ...Getting Tweets for RepPerlmutter
    Lib_tweets len: 287400
    ...Getting Tweets for SenGaryPeters
    Lib_tweets len: 290617
    ...Getting Tweets for collinpeterson
    Lib_tweets len: 290816
    ...Getting Tweets for ChelliePingree
    Lib_tweets len: 294023
    ...Getting Tweets for RepDavidEPrice
    Lib_tweets len: 297226
    ...Getting Tweets for RepMikeQuigley
    Lib_tweets len: 300435
    ...Getting Tweets for RepRichmond
    Lib_tweets len: 303033
    ...Getting Tweets for RepRoybalAllard
    Lib_tweets len: 306261
    ...Getting Tweets for Call_Me_Dutch
    Lib_tweets len: 309503
    ...Getting Tweets for RepBobbyRush
    Lib_tweets len: 312708
    ...Getting Tweets for RepTimRyan
    Lib_tweets len: 315928
    ...Getting Tweets for Kilili_Sablan
    Lib_tweets len: 317335
    ...Getting Tweets for RepSarbanes
    Lib_tweets len: 319807
    ...Getting Tweets for JanSchakowsky
    Lib_tweets len: 323056
    ...Getting Tweets for RepAdamSchiff
    Lib_tweets len: 326281
    ...Getting Tweets for RepSchrader
    Lib_tweets len: 328838
    ...Getting Tweets for SenSchumer
    Lib_tweets len: 332082
    ...Getting Tweets for RepDavidScott
    Lib_tweets len: 334974
    ...Getting Tweets for BobbyScott
    Lib_tweets len: 338195
    ...Getting Tweets for RepJoseSerrano
    Lib_tweets len: 341432
    ...Getting Tweets for RepTerriSewell
    Lib_tweets len: 344680
    ...Getting Tweets for BradSherman
    Lib_tweets len: 346988
    ...Getting Tweets for RepSires
    Lib_tweets len: 350192
    ...Getting Tweets for RepAdamSmith
    Lib_tweets len: 353400
    ...Getting Tweets for RepSpeier
    Lib_tweets len: 356610
    ...Getting Tweets for RepLindaSanchez
    Lib_tweets len: 359840
    ...Getting Tweets for BennieGThompson
    Lib_tweets len: 362296
    ...Getting Tweets for RepThompson
    Lib_tweets len: 365546
    ...Getting Tweets for RepPaulTonko
    Lib_tweets len: 368743
    ...Getting Tweets for ChrisVanHollen
    Lib_tweets len: 371968
    ...Getting Tweets for NydiaVelazquez
    Lib_tweets len: 375205
    ...Getting Tweets for RepVisclosky
    Lib_tweets len: 377138
    ...Getting Tweets for RepDWStweets
    Lib_tweets len: 380374
    ...Getting Tweets for RepMaxineWaters
    Lib_tweets len: 383414
    ...Getting Tweets for PeterWelch
    Lib_tweets len: 385797
    ...Getting Tweets for RepWilson
    Lib_tweets len: 388992
    ...Getting Tweets for RonWyden
    Lib_tweets len: 392237
    ...Getting Tweets for RepJohnYarmuth
    Lib_tweets len: 395448
    ...Getting Tweets for RepBonamici
    Lib_tweets len: 398684
    ...Getting Tweets for RepDelBene
    Lib_tweets len: 401927
    ...Getting Tweets for RepDonaldPayne
    Lib_tweets len: 405158
    ...Getting Tweets for SenBrianSchatz
    Lib_tweets len: 406816
    ...Getting Tweets for RepBillFoster
    Lib_tweets len: 410039
    ...Getting Tweets for RepDinaTitus
    Lib_tweets len: 413252
    ...Getting Tweets for SenatorSinema
    Lib_tweets len: 416479
    ...Getting Tweets for RepHuffman
    Lib_tweets len: 419356
    ...Getting Tweets for RepBera
    Lib_tweets len: 422555
    ...Getting Tweets for RepSwalwell
    Lib_tweets len: 425770
    ...Getting Tweets for RepBrownley
    Lib_tweets len: 429004
    ...Getting Tweets for RepCardenas
    Lib_tweets len: 432195
    ...Getting Tweets for CongressmanRuiz
    Lib_tweets len: 434046
    ...Getting Tweets for RepMarkTakano
    Lib_tweets len: 437280
    ...Getting Tweets for RepLowenthal
    Lib_tweets len: 440491
    ...Getting Tweets for RepJuanVargas
    Lib_tweets len: 442911
    ...Getting Tweets for RepScottPeters
    Lib_tweets len: 446130
    ...Getting Tweets for RepLoisFrankel
    Lib_tweets len: 449343
    ...Getting Tweets for TulsiPress
    Lib_tweets len: 452569
    ...Getting Tweets for SenDuckworth
    Lib_tweets len: 455776
    ...Getting Tweets for RepCheri
    Lib_tweets len: 459013
    ...Getting Tweets for SenWarren
    Lib_tweets len: 462232
    ...Getting Tweets for RepJoeKennedy
    Lib_tweets len: 465470
    ...Getting Tweets for SenAngusKing
    Lib_tweets len: 468703
    ...Getting Tweets for RepDanKildee
    Lib_tweets len: 471903
    ...Getting Tweets for RepAnnieKuster
    Lib_tweets len: 475106
    ...Getting Tweets for RepGraceMeng
    Lib_tweets len: 478318
    ...Getting Tweets for RepJeffries
    Lib_tweets len: 481541
    ...Getting Tweets for RepSeanMaloney
    Lib_tweets len: 484785
    ...Getting Tweets for RepBeatty
    Lib_tweets len: 488023
    ...Getting Tweets for RepCartwright
    Lib_tweets len: 490243
    ...Getting Tweets for JoaquinCastrotx
    Lib_tweets len: 493463
    ...Getting Tweets for RepVeasey
    Lib_tweets len: 496672
    ...Getting Tweets for RepFilemonVela
    Lib_tweets len: 498193
    ...Getting Tweets for timkaine
    Lib_tweets len: 501431
    ...Getting Tweets for RepDerekKilmer
    Lib_tweets len: 504647
    ...Getting Tweets for RepDennyHeck
    Lib_tweets len: 507879
    ...Getting Tweets for RepMarkPocan
    Lib_tweets len: 511091
    ...Getting Tweets for RepRobinKelly
    Lib_tweets len: 514326
    ...Getting Tweets for SenBooker
    Lib_tweets len: 517539
    ...Getting Tweets for RepKClark
    Lib_tweets len: 520742
    ...Getting Tweets for DonaldNorcross
    Lib_tweets len: 523950
    ...Getting Tweets for RepAdams
    Lib_tweets len: 527194
    ...Getting Tweets for RepRubenGallego
    Lib_tweets len: 530407
    ...Getting Tweets for RepDeSaulnier
    Lib_tweets len: 533617
    ...Getting Tweets for reppeteaguilar
    Lib_tweets len: 535365
    ...Getting Tweets for RepTedLieu
    Lib_tweets len: 538596
    ...Getting Tweets for NormaJTorres
    Lib_tweets len: 541820
    ...Getting Tweets for teammoulton
    Lib_tweets len: 545025
    ...Getting Tweets for RepDebDingell
    Lib_tweets len: 548256
    ...Getting Tweets for RepLawrence
    Lib_tweets len: 551469
    ...Getting Tweets for RepBonnie
    Lib_tweets len: 554708
    ...Getting Tweets for RepKathleenRice
    Lib_tweets len: 557933
    ...Getting Tweets for CongBoyle
    Lib_tweets len: 561136
    ...Getting Tweets for RepDonBeyer
    Lib_tweets len: 564337
    ...Getting Tweets for staceyplaskett
    Lib_tweets len: 566209
    ...Getting Tweets for RepDwightEvans
    Lib_tweets len: 569436
    ...Getting Tweets for SenKamalaHarris
    Lib_tweets len: 572657
    ...Getting Tweets for Senatorhassan
    Lib_tweets len: 575875
    ...Getting Tweets for sencortezmasto
    Lib_tweets len: 579103
    ...Getting Tweets for repschneider
    Lib_tweets len: 582340
    ...Getting Tweets for repohalleran
    Lib_tweets len: 585552
    ...Getting Tweets for RepRoKhanna
    Lib_tweets len: 588793
    ...Getting Tweets for RepJimmyPanetta
    Lib_tweets len: 591449
    ...Getting Tweets for RepCarbajal
    Lib_tweets len: 594185
    ...Getting Tweets for RepBarragan
    Lib_tweets len: 597399
    ...Getting Tweets for reploucorrea
    Lib_tweets len: 598056
    ...Getting Tweets for RepLBR
    Lib_tweets len: 599192
    ...Getting Tweets for RepAlLawsonJr
    Lib_tweets len: 600479
    ...Getting Tweets for RepStephMurphy
    Lib_tweets len: 603373
    ...Getting Tweets for RepDarrenSoto
    Lib_tweets len: 606599
    ...Getting Tweets for RepValDemings
    Lib_tweets len: 609807
    ...Getting Tweets for repcharliecrist
    Lib_tweets len: 612440
    ...Getting Tweets for congressmanraja
    Lib_tweets len: 615662
    ...Getting Tweets for RepAnthonyBrown
    Lib_tweets len: 618875
    ...Getting Tweets for repraskin
    Lib_tweets len: 622116
    ...Getting Tweets for RepJoshG
    Lib_tweets len: 625360
    ...Getting Tweets for SenJackyRosen
    Lib_tweets len: 628587
    ...Getting Tweets for RepTomSuozzi
    Lib_tweets len: 629710
    ...Getting Tweets for RepEspaillat
    Lib_tweets len: 632947
    ...Getting Tweets for RepGonzalez
    Lib_tweets len: 634391
    ...Getting Tweets for RepMcEachin
    Lib_tweets len: 637611
    ...Getting Tweets for RepJayapal
    Lib_tweets len: 640824
    ...Getting Tweets for RepJimmyGomez
    Lib_tweets len: 644044
    ...Getting Tweets for sendougjones
    Lib_tweets len: 645139
    ...Getting Tweets for SenTinaSmith
    Lib_tweets len: 648057
    ...Getting Tweets for RepConorLamb
    Lib_tweets len: 648444
    ...Getting Tweets for RepJoeMorelle
    Lib_tweets len: 649844
    ...Getting Tweets for RepMGS
    Lib_tweets len: 652451
    ...Getting Tweets for RepSusanWild
    Lib_tweets len: 653814
    ...Getting Tweets for RepEdCase
    Lib_tweets len: 654362
    ...Getting Tweets for RepHorsford
    Lib_tweets len: 657581
    ...Getting Tweets for RepKirkpatrick
    Lib_tweets len: 658699
    ...Getting Tweets for RepGregStanton
    Lib_tweets len: 659200
    ...Getting Tweets for RepJoshHarder
    Lib_tweets len: 660196
    ...Getting Tweets for RepTjCox
    Lib_tweets len: 661198
    ...Getting Tweets for RepGilCisneros
    Lib_tweets len: 662509
    ...Getting Tweets for RepKatiePorter
    Lib_tweets len: 663500
    ...Getting Tweets for RepHarley
    Lib_tweets len: 664278
    ...Getting Tweets for RepMikeLevin
    Lib_tweets len: 665109
    ...Getting Tweets for RepJoeNeguse
    Lib_tweets len: 667358
    ...Getting Tweets for RepJasonCrow
    Lib_tweets len: 668599
    ...Getting Tweets for RepJahanaHayes
    Lib_tweets len: 670011
    ...Getting Tweets for RepDMP
    Lib_tweets len: 672947
    ...Getting Tweets for RepShalala
    Lib_tweets len: 674518
    ...Getting Tweets for replucymcbath
    Lib_tweets len: 675001
    ...Getting Tweets for GuamCongressman
    Lib_tweets len: 675047
    ...Getting Tweets for RepFinkenauer
    Lib_tweets len: 675525
    ...Getting Tweets for RepCindyAxne
    Lib_tweets len: 676335
    ...Getting Tweets for RepChuyGarcia
    Lib_tweets len: 677615
    ...Getting Tweets for RepCasten
    Lib_tweets len: 678949
    ...Getting Tweets for RepUnderwood
    Lib_tweets len: 679803
    ...Getting Tweets for RepDavids
    Lib_tweets len: 680588
    ...Getting Tweets for RepLoriTrahan
    Lib_tweets len: 683305
    ...Getting Tweets for RepPressley
    Lib_tweets len: 685264
    ...Getting Tweets for repdavidtrone
    Lib_tweets len: 686408
    ...Getting Tweets for RepSlotkin
    Lib_tweets len: 687034
    ...Getting Tweets for RepAndyLevin
    Lib_tweets len: 687995
    ...Getting Tweets for RepHaleyStevens
    Lib_tweets len: 689458
    ...Getting Tweets for RepRashida
    Lib_tweets len: 690076
    ...Getting Tweets for RepAngieCraig
    Lib_tweets len: 691152
    ...Getting Tweets for RepDeanPhillips
    Lib_tweets len: 692164
    ...Getting Tweets for Ilhan
    Lib_tweets len: 693897
    ...Getting Tweets for RepChrisPappas
    Lib_tweets len: 694972
    ...Getting Tweets for RepAndyKimNJ
    Lib_tweets len: 695704
    ...Getting Tweets for RepMalinowski
    Lib_tweets len: 696117
    ...Getting Tweets for RepSherrill
    Lib_tweets len: 697325
    ...Getting Tweets for RepDebHaaland
    Lib_tweets len: 699689
    ...Getting Tweets for RepTorresSmall
    Lib_tweets len: 700146
    ...Getting Tweets for RepSusieLee
    Lib_tweets len: 702775
    ...Getting Tweets for RepMaxRose
    Lib_tweets len: 703632
    ...Getting Tweets for RepAOC
    Lib_tweets len: 703713
    ...Getting Tweets for repdelgado
    Lib_tweets len: 705245
    ...Getting Tweets for RepBrindisi
    Lib_tweets len: 706519
    ...Getting Tweets for RepKendraHorn
    Lib_tweets len: 707116
    ...Getting Tweets for RepDean
    Lib_tweets len: 710093
    ...Getting Tweets for RepHoulahan
    Lib_tweets len: 711271
    ...Getting Tweets for RepCunningham
    Lib_tweets len: 712199
    ...Getting Tweets for RepFletcher
    Lib_tweets len: 713236
    ...Getting Tweets for RepEscobar
    Lib_tweets len: 716030
    ...Getting Tweets for RepSylviaGarcia
    Lib_tweets len: 717841
    ...Getting Tweets for RepColinAllred
    Lib_tweets len: 718719
    ...Getting Tweets for RepBenMcAdams
    Lib_tweets len: 721957
    ...Getting Tweets for RepElaineLuria
    Lib_tweets len: 723488
    ...Getting Tweets for RepSpanberger
    Lib_tweets len: 725402
    ...Getting Tweets for RepWexton
    Lib_tweets len: 727137
    ...Getting Tweets for RepKimSchrier
    Lib_tweets len: 727664
    ...Getting Tweets for repgolden
    Lib_tweets len: 728175



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

    ...Getting Tweets for SenJohnBarrasso
    con_tweets len: 3209
    ...Getting Tweets for SenatorWicker
    con_tweets len: 6408
    ...Getting Tweets for SenAlexander
    con_tweets len: 9637
    ...Getting Tweets for SenatorCollins
    con_tweets len: 12880
    ...Getting Tweets for JohnCornyn
    con_tweets len: 16095
    ...Getting Tweets for SenatorEnzi
    con_tweets len: 19302
    ...Getting Tweets for GrahamBlog
    con_tweets len: 22540
    ...Getting Tweets for InhofePress
    con_tweets len: 25736
    ...Getting Tweets for McConnellPress
    con_tweets len: 28943
    ...Getting Tweets for SenatorRisch
    con_tweets len: 30786
    ...Getting Tweets for SenPatRoberts
    con_tweets len: 34010
    ...Getting Tweets for Robert_Aderholt
    con_tweets len: 35779
    ...Getting Tweets for justinamash
    con_tweets len: 38999
    ...Getting Tweets for RepGusBilirakis
    con_tweets len: 42207
    ...Getting Tweets for RepRobBishop
    problem with RepRobBishop in loop
    ...Getting Tweets for MarshaBlackburn
    con_tweets len: 45453
    ...Getting Tweets for RoyBlunt
    con_tweets len: 48699
    ...Getting Tweets for JohnBoozman
    con_tweets len: 51914
    ...Getting Tweets for RepKevinBrady
    con_tweets len: 55163
    ...Getting Tweets for RepMoBrooks
    con_tweets len: 58244
    ...Getting Tweets for VernBuchanan
    con_tweets len: 60992
    ...Getting Tweets for RepLarryBucshon
    con_tweets len: 64217
    ...Getting Tweets for MichaelCBurgess
    con_tweets len: 67427
    ...Getting Tweets for SenatorBurr
    con_tweets len: 70670
    ...Getting Tweets for KenCalvert
    con_tweets len: 73895
    ...Getting Tweets for SenCapito
    con_tweets len: 77124
    ...Getting Tweets for JudgeCarter
    con_tweets len: 80371
    ...Getting Tweets for SenBillCassidy
    con_tweets len: 82070
    ...Getting Tweets for RepSteveChabot
    con_tweets len: 85255
    ...Getting Tweets for TomColeOK04
    con_tweets len: 86880
    ...Getting Tweets for ConawayTX11
    con_tweets len: 87947
    ...Getting Tweets for MikeCrapo
    con_tweets len: 91159
    ...Getting Tweets for RepRickCrawford
    con_tweets len: 94377
    ...Getting Tweets for DesJarlaisTN04
    con_tweets len: 97589
    ...Getting Tweets for MarioDB
    con_tweets len: 100798
    ...Getting Tweets for RepJeffDuncan
    con_tweets len: 104027
    ...Getting Tweets for RepChuck
    con_tweets len: 107056
    ...Getting Tweets for RepBillFlores
    con_tweets len: 110288
    ...Getting Tweets for JeffFortenberry
    con_tweets len: 113501
    ...Getting Tweets for VirginiaFoxx
    con_tweets len: 116741
    ...Getting Tweets for SenCoryGardner
    con_tweets len: 119958
    ...Getting Tweets for RepBobGibbs
    con_tweets len: 123162
    ...Getting Tweets for RepLouieGohmert
    con_tweets len: 126399
    ...Getting Tweets for RepGosar
    con_tweets len: 129606
    ...Getting Tweets for RepKayGranger
    con_tweets len: 131633
    ...Getting Tweets for ChuckGrassley
    con_tweets len: 134850
    ...Getting Tweets for RepSamGraves
    con_tweets len: 138016
    ...Getting Tweets for RepTomGraves
    con_tweets len: 141236
    ...Getting Tweets for RepMGriffith
    con_tweets len: 142720
    ...Getting Tweets for RepGuthrie
    con_tweets len: 143596
    ...Getting Tweets for RepAndyHarrisMD
    con_tweets len: 146825
    ...Getting Tweets for RepHartzler
    con_tweets len: 150025
    ...Getting Tweets for HerreraBeutler
    con_tweets len: 151433
    ...Getting Tweets for SenJohnHoeven
    con_tweets len: 154682
    ...Getting Tweets for RepHuizenga
    con_tweets len: 157915
    ...Getting Tweets for Rep_Hunter
    error with Rep_Hunter in function
    problem with Rep_Hunter in loop
    ...Getting Tweets for RepBillJohnson
    con_tweets len: 161155
    ...Getting Tweets for SenRonJohnson
    con_tweets len: 164391
    ...Getting Tweets for Jim_Jordan
    con_tweets len: 167614
    ...Getting Tweets for MikeKellyPA
    con_tweets len: 170385
    ...Getting Tweets for RepPeteKing
    con_tweets len: 173382
    ...Getting Tweets for SteveKingIA
    con_tweets len: 176334
    ...Getting Tweets for RepKinzinger
    con_tweets len: 179547
    ...Getting Tweets for RepDLamborn
    con_tweets len: 182764
    ...Getting Tweets for SenatorLankford
    con_tweets len: 185977
    ...Getting Tweets for BobLatta
    con_tweets len: 189182
    ...Getting Tweets for SenMikeLee
    con_tweets len: 192422
    ...Getting Tweets for USRepLong
    con_tweets len: 195632
    ...Getting Tweets for RepFrankLucas
    con_tweets len: 196742
    ...Getting Tweets for RepBlaine
    con_tweets len: 198863
    ...Getting Tweets for RepKenMarchant
    con_tweets len: 202107
    ...Getting Tweets for GOPLeader
    con_tweets len: 205324
    ...Getting Tweets for RepMcCaul
    con_tweets len: 208531
    ...Getting Tweets for RepMcClintock
    con_tweets len: 209086
    ...Getting Tweets for PatrickMcHenry
    con_tweets len: 212323
    ...Getting Tweets for RepMcKinley
    con_tweets len: 215532
    ...Getting Tweets for CathyMcMorris
    con_tweets len: 218753
    ...Getting Tweets for JerryMoran
    con_tweets len: 221963
    ...Getting Tweets for LisaMurkowski
    con_tweets len: 225199
    ...Getting Tweets for RepDevinNunes
    con_tweets len: 225381
    ...Getting Tweets for RepPeteOlson
    con_tweets len: 228626
    ...Getting Tweets for CongPalazzo
    con_tweets len: 231859
    ...Getting Tweets for RandPaul
    con_tweets len: 235108
    ...Getting Tweets for SenRobPortman
    con_tweets len: 238348
    ...Getting Tweets for CongBillPosey
    con_tweets len: 239652
    ...Getting Tweets for RepTomReed
    con_tweets len: 242886
    ...Getting Tweets for RepMarthaRoby
    con_tweets len: 246125
    ...Getting Tweets for DrPhilRoe
    con_tweets len: 249337
    ...Getting Tweets for RepHalRogers
    con_tweets len: 252568
    ...Getting Tweets for RepMikeRogersAL
    con_tweets len: 255815
    ...Getting Tweets for SenRubioPress
    con_tweets len: 259042
    ...Getting Tweets for SteveScalise
    con_tweets len: 262267
    ...Getting Tweets for RepDavid
    con_tweets len: 265505
    ...Getting Tweets for AustinScottGA08
    con_tweets len: 268506
    ...Getting Tweets for SenatorTimScott
    con_tweets len: 271740
    ...Getting Tweets for JimPressOffice
    con_tweets len: 273417
    ...Getting Tweets for SenShelby
    con_tweets len: 275870
    ...Getting Tweets for RepShimkus
    con_tweets len: 279090
    ...Getting Tweets for CongMikeSimpson
    con_tweets len: 280316
    ...Getting Tweets for RepAdrianSmith
    con_tweets len: 283534
    ...Getting Tweets for RepChrisSmith
    con_tweets len: 284654
    ...Getting Tweets for RepSteveStivers
    con_tweets len: 287894
    ...Getting Tweets for CongressmanGT
    con_tweets len: 291084
    ...Getting Tweets for MacTXPress
    con_tweets len: 292361
    ...Getting Tweets for SenJohnThune
    con_tweets len: 295591
    ...Getting Tweets for RepTipton
    con_tweets len: 298835
    ...Getting Tweets for SenToomey
    con_tweets len: 302044
    ...Getting Tweets for RepMikeTurner
    con_tweets len: 304943
    ...Getting Tweets for RepFredUpton
    con_tweets len: 308182
    ...Getting Tweets for RepWalberg
    con_tweets len: 311384
    ...Getting Tweets for RepGregWalden
    con_tweets len: 314349
    ...Getting Tweets for RepWebster
    con_tweets len: 317570
    ...Getting Tweets for RepJoeWilson
    con_tweets len: 320377
    ...Getting Tweets for RobWittman
    con_tweets len: 323578
    ...Getting Tweets for Rep_SteveWomack
    con_tweets len: 326804
    ...Getting Tweets for RepRobWoodall
    con_tweets len: 328120
    ...Getting Tweets for RepDonYoung
    con_tweets len: 330050
    ...Getting Tweets for SenToddYoung
    con_tweets len: 333272
    ...Getting Tweets for MarkAmodeiNV2
    con_tweets len: 334516
    ...Getting Tweets for RepThomasMassie
    con_tweets len: 337759
    ...Getting Tweets for SenTomCotton
    con_tweets len: 341004
    ...Getting Tweets for RepLaMalfa
    con_tweets len: 342721
    ...Getting Tweets for RepPaulCook
    con_tweets len: 344291
    ...Getting Tweets for RepTedYoho
    con_tweets len: 345878
    ...Getting Tweets for RepDougCollins
    con_tweets len: 349114
    ...Getting Tweets for RodneyDavis
    con_tweets len: 352362
    ...Getting Tweets for RepWalorski
    con_tweets len: 355603
    ...Getting Tweets for SusanWBrooks
    con_tweets len: 358831
    ...Getting Tweets for RepAndyBarr
    con_tweets len: 362041
    ...Getting Tweets for RepAnnWagner
    con_tweets len: 365284
    ...Getting Tweets for SteveDaines
    con_tweets len: 368525
    ...Getting Tweets for RepRichHudson
    con_tweets len: 371731
    ...Getting Tweets for RepMarkMeadows
    con_tweets len: 374463
    ...Getting Tweets for RepHolding
    con_tweets len: 375405
    ...Getting Tweets for SenKevinCramer
    con_tweets len: 378640
    ...Getting Tweets for SenatorFischer
    con_tweets len: 381871
    ...Getting Tweets for RepBradWenstrup
    con_tweets len: 385096
    ...Getting Tweets for RepDaveJoyce
    con_tweets len: 388304
    ...Getting Tweets for RepMullin
    con_tweets len: 391549
    ...Getting Tweets for RepScottPerry
    con_tweets len: 394777
    ...Getting Tweets for RepTomRice
    con_tweets len: 396877
    ...Getting Tweets for SenTedCruz
    con_tweets len: 400107
    ...Getting Tweets for TXRandy14
    con_tweets len: 403207
    ...Getting Tweets for RepRWilliams
    con_tweets len: 406438
    ...Getting Tweets for RepChrisStewart
    con_tweets len: 409662
    ...Getting Tweets for RepJasonSmith
    con_tweets len: 412867
    ...Getting Tweets for RepByrne
    con_tweets len: 416091
    ...Getting Tweets for USRepGaryPalmer
    con_tweets len: 418284
    ...Getting Tweets for RepFrenchHill
    con_tweets len: 421305
    ...Getting Tweets for RepWesterman
    con_tweets len: 424514
    ...Getting Tweets for RepKenBuck
    con_tweets len: 426290
    ...Getting Tweets for RepBuddyCarter
    con_tweets len: 428335
    ...Getting Tweets for congressmanhice
    con_tweets len: 431541
    ...Getting Tweets for RepLoudermilk
    con_tweets len: 434063
    ...Getting Tweets for reprickallen
    con_tweets len: 435865
    ...Getting Tweets for RepBost
    con_tweets len: 437997
    ...Getting Tweets for RepAbraham
    con_tweets len: 439042
    ...Getting Tweets for RepGarretGraves
    con_tweets len: 442238
    ...Getting Tweets for RepMoolenaar
    con_tweets len: 443092
    ...Getting Tweets for RepTomEmmer
    con_tweets len: 446322
    ...Getting Tweets for RepDavidRouzer
    con_tweets len: 447532
    ...Getting Tweets for RepLeeZeldin
    con_tweets len: 450484
    ...Getting Tweets for RepStefanik
    con_tweets len: 453715
    ...Getting Tweets for RepJohnKatko
    con_tweets len: 454928
    ...Getting Tweets for RepRatcliffe
    con_tweets len: 457643
    ...Getting Tweets for hurdonthehill
    con_tweets len: 460859
    ...Getting Tweets for RepBrianBabin
    con_tweets len: 463886
    ...Getting Tweets for RepNewhouse
    con_tweets len: 465674
    ...Getting Tweets for RepGrothman
    con_tweets len: 468057
    ...Getting Tweets for RepAlexMooney
    con_tweets len: 470010
    ...Getting Tweets for RepAmata
    con_tweets len: 470476
    ...Getting Tweets for SenDanSullivan
    con_tweets len: 473297
    ...Getting Tweets for sendavidperdue
    con_tweets len: 476537
    ...Getting Tweets for SenJoniErnst
    con_tweets len: 479748
    ...Getting Tweets for senthomtillis
    con_tweets len: 482949
    ...Getting Tweets for SenatorRounds
    con_tweets len: 486150
    ...Getting Tweets for RepMarkWalker
    con_tweets len: 489358
    ...Getting Tweets for SenSasse
    con_tweets len: 490267
    ...Getting Tweets for reptrentkelly
    con_tweets len: 492022
    ...Getting Tweets for RepLaHood
    con_tweets len: 494010
    ...Getting Tweets for WarrenDavidson
    con_tweets len: 497136
    ...Getting Tweets for KYComer
    con_tweets len: 500301
    ...Getting Tweets for SenJohnKennedy
    con_tweets len: 502215
    ...Getting Tweets for RepAndyBiggsAZ
    con_tweets len: 505435
    ...Getting Tweets for RepMattGaetz
    con_tweets len: 508671
    ...Getting Tweets for drnealdunnfl2
    con_tweets len: 510368
    ...Getting Tweets for RepRutherfordFL
    con_tweets len: 511636
    ...Getting Tweets for repbrianmast
    con_tweets len: 514137
    ...Getting Tweets for RepRooney
    con_tweets len: 515112
    ...Getting Tweets for RepDrewFerguson
    con_tweets len: 516837
    ...Getting Tweets for RepJimBanks
    con_tweets len: 520072
    ...Getting Tweets for reptrey
    con_tweets len: 520811
    ...Getting Tweets for RepMarshall
    problem with RepMarshall in loop
    ...Getting Tweets for RepClayHiggins
    con_tweets len: 521974
    ...Getting Tweets for RepMikeJohnson
    con_tweets len: 524184
    ...Getting Tweets for RepJackBergman
    con_tweets len: 525284
    ...Getting Tweets for RepPaulMitchell
    con_tweets len: 527037
    ...Getting Tweets for RepTedBudd
    con_tweets len: 528870
    ...Getting Tweets for repdonbacon
    con_tweets len: 532077
    ...Getting Tweets for repbrianfitz
    con_tweets len: 533545
    ...Getting Tweets for RepSmucker
    con_tweets len: 536761
    ...Getting Tweets for repjenniffer
    con_tweets len: 539992
    ...Getting Tweets for repdavidkustoff
    con_tweets len: 541585
    ...Getting Tweets for RepArrington
    con_tweets len: 543774
    ...Getting Tweets for RepGallagher
    con_tweets len: 546541
    ...Getting Tweets for RepLizCheney
    con_tweets len: 547124
    ...Getting Tweets for RepRonEstes
    con_tweets len: 548316
    ...Getting Tweets for GregForMontana
    con_tweets len: 549560
    ...Getting Tweets for RepRalphNorman
    con_tweets len: 551257
    ...Getting Tweets for RepJohnCurtis
    con_tweets len: 553362
    ...Getting Tweets for SenHydeSmith
    con_tweets len: 554418
    ...Getting Tweets for RepDLesko
    con_tweets len: 556289
    ...Getting Tweets for RepCloudTX
    con_tweets len: 556451
    ...Getting Tweets for RepBalderson
    con_tweets len: 557004
    ...Getting Tweets for repkevinhern
    con_tweets len: 557310
    ...Getting Tweets for RepMichaelWaltz
    con_tweets len: 559011
    ...Getting Tweets for RepRossSpano
    con_tweets len: 559551
    ...Getting Tweets for RepGregSteube
    con_tweets len: 559974
    ...Getting Tweets for RepRussFulcher
    con_tweets len: 560461
    ...Getting Tweets for RepJimBaird
    con_tweets len: 560694
    ...Getting Tweets for RepGregPence
    con_tweets len: 561325
    ...Getting Tweets for Rep_Watkins
    con_tweets len: 561955
    ...Getting Tweets for RepHagedorn
    con_tweets len: 562591
    ...Getting Tweets for RepPeteStauber
    con_tweets len: 563049
    ...Getting Tweets for RepMichaelGuest
    con_tweets len: 563556
    ...Getting Tweets for RepArmstrongND
    con_tweets len: 564441
    ...Getting Tweets for CongressmanJVD
    con_tweets len: 564873
    ...Getting Tweets for RepAGonzalez
    con_tweets len: 565440
    ...Getting Tweets for RepMeuser
    con_tweets len: 566022
    ...Getting Tweets for RepJohnJoyce
    con_tweets len: 566744
    ...Getting Tweets for GReschenthaler
    con_tweets len: 568150
    ...Getting Tweets for RepTimmons
    con_tweets len: 568355
    ...Getting Tweets for RepDustyJohnson
    con_tweets len: 568996
    ...Getting Tweets for RepTimBurchett
    con_tweets len: 569444
    ...Getting Tweets for RepJohnRose
    con_tweets len: 569815
    ...Getting Tweets for RepMarkGreen
    con_tweets len: 571002
    ...Getting Tweets for RepDanCrenshaw
    con_tweets len: 571610
    ...Getting Tweets for RepVanTaylor
    con_tweets len: 571963
    ...Getting Tweets for RepLanceGooden
    error with RepLanceGooden in function
    problem with RepLanceGooden in loop
    ...Getting Tweets for RepRonWright
    con_tweets len: 572511
    ...Getting Tweets for RepChipRoy
    con_tweets len: 574533
    ...Getting Tweets for RepRiggleman
    con_tweets len: 575672
    ...Getting Tweets for RepBenCline
    con_tweets len: 576084
    ...Getting Tweets for RepBryanSteil
    con_tweets len: 577288
    ...Getting Tweets for RepCarolMiller
    con_tweets len: 577802
    ...Getting Tweets for SenRickScott
    con_tweets len: 581012
    ...Getting Tweets for SenatorBraun
    con_tweets len: 581878
    ...Getting Tweets for SenHawleyPress
    con_tweets len: 582879
    ...Getting Tweets for SenatorRomney
    con_tweets len: 583455
    ...Getting Tweets for SenMcSallyAZ
    con_tweets len: 586701
    ...Getting Tweets for RepFredKeller
    con_tweets len: 587295
    ...Getting Tweets for jdanbishop
    con_tweets len: 589105
    ...Getting Tweets for DrGregMurphy1
    con_tweets len: 589158
    ...Getting Tweets for SenatorLoeffler
    con_tweets len: 589235


### Manually fix the problems that arose for a few Conservative Handles


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


```python

```
