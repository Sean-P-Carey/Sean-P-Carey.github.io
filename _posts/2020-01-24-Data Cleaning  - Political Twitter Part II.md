---
Title: "Data Cleaning  - Political Twitter Part II"
date: 2020-01-24
tags: [twitter, dask, spaCy, NLTK, NLP, practicum]
header:
   image: "/images/background.jpg"
excerpt: "Perform basic NLP Text Preprocessing using REGEX, NLTK and, spaCy"
---

## Data Cleaning


This post will take the data that was created in [Part 1](https://sean-p-carey.github.io/data_collection/) and perform normal text data preprocessing.  

**Begin by removing items from the text that are not needed because they will add no value to the classification model**

* URLs  
* hashtags and Twitter @usernames  
* Emoticons.
* Punctuation  

**Next we perform some more common NLP Preprocessing tasks:**

* Tokenization
* Removal of Stopwords  
* Lemmatization

***


```python
import pandas as pd

import numpy as np
import os
import pickle
import boto3
s3 = boto3.resource('s3')
bucket_name = "msds-practicum-carey"

import re
import spacy
#nlp = spacy.load("en_core_web_sm")

nlp = spacy.load('en_core_web_sm', disable=['ner', 'parser'])
#nlp.add_pipe(nlp.create_pipe('sentencizer'))

import nltk
from nltk import FreqDist
import string

import warnings
warnings.filterwarnings('ignore')

```

## Download the Data from AWS S3.

To keep the size of the code repository small I have stored all of the data in an S3 Object Store. The other option would be to use GIT LFS. All of the intermediate data has been store as a .pkl (pickle) file. This is a convenient way to serialize any variable from python in a portable way.  

tweet_df.pkl is a serialized Pandas Dataframe

### Send the updated Dataframe to Amazon S3.



```python
with open('outdata/tweet_df.pkl', 'wb') as data:
    s3.Bucket(bucket_name).download_fileobj('tweet_df.pkl', data)

tweet_df = pd.read_pickle('outdata/tweet_df.pkl')

os.remove('outdata/tweet_df.pkl')
```


```python
tweet_df.sample(10)
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
      <th>1056834</th>
      <td>Homeland Security Committee hearing on #TSA &amp;a...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>613094</th>
      <td>The last major US nail manufacturer--and their...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>405118</th>
      <td>In case you missed it, @SpeakerRyan, @RepKevin...</td>
      <td>C</td>
    </tr>
    <tr>
      <th>1224362</th>
      <td>VP Biden is on his last stop of the day in Ohi...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>643432</th>
      <td>I fundamentally refuse to let Americans pay mo...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>842083</th>
      <td>RT @JasonKander: When I was Secretary of State...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>1213864</th>
      <td>RT @cppj: Stay up-to-date on state road and hi...</td>
      <td>C</td>
    </tr>
    <tr>
      <th>500238</th>
      <td>RT @NJTVNews: .@SenatorMenendez calls for $2.5...</td>
      <td>L</td>
    </tr>
    <tr>
      <th>599936</th>
      <td>RT @DonDaileyAPT: Tonight @ 8 on @CapitolJourn...</td>
      <td>C</td>
    </tr>
    <tr>
      <th>284711</th>
      <td>RT @HouseJudiciary: Statement from @GOPLeader,...</td>
      <td>C</td>
    </tr>
  </tbody>
</table>
</div>



## Import Stopwords from NLTK and define text cleaning functions.

NLTK keeps a library of "stopwords". Thesea are words that will typically show up the most in a text but will add very little substance to the analysis. Exampels of STOPWORDS are: "THE", "AN", "a" etc...

We can also add words to the list of stopwords. This is done on a project by project basis dependent upon the origin of the text. In our case the corpus came from Twitter so we know a good portion of it will start with "RT" which stands for "retweet". It adds nothing to the analysis so we will add it to the list of stopwords.


```python
# import stopwords
stopwords = nltk.corpus.stopwords.words('english')
stopwords.extend(['RT'])
```


```python

# breaks text up in to a list of individual words
def tokenize(text):

    tokens = nltk.word_tokenize(text)

    return tokens

# removes stopwords
def remove_stopwords(words):


    filtered = filter(lambda word: word not in stopwords, words)

    return list(filtered)

#  lemmatizes text based on the part of speech tags
def lemmatize(text, nlp=nlp):

    doc = nlp(text)

    lemmatized = [token.lemma_ for token in doc]

    return " ".join(lemmatized)

# applies the lemmatize function to a dataframe
# allows us to use Dask to run function in parallel
def clean_text(df):

    df["clean_tweets"] = [lemmatize(x) for x in df['clean_tweets'].tolist()]
    print('done')
    return df

# Gets rid of emojis and some oddly formated strings
def remove_emoji(inputString):
    return inputString.encode('ascii', 'ignore').decode('ascii')
```

## Use REGEX and the defined functions to perform  preprocessing.

### 1. Remove URLs


```python
tweet_df['clean_tweets'] =\
tweet_df['tweet'].apply(lambda x: re.sub('http://\S+','', x))

tweet_df['clean_tweets'] =\
tweet_df['clean_tweets'].apply(lambda x: re.sub('https://\S+', '', x))
```

### 2. Remove @name mentions and Emojis


```python
tweet_df['clean_tweets'] =\
tweet_df['clean_tweets'].apply(lambda x: re.sub('@\S+', '', x))

tweet_df['clean_tweets'] =\
tweet_df['clean_tweets'].apply(lambda x: remove_emoji(x))
```

### 3. Remove new line Characters


```python
tweet_df['clean_tweets'] =\
tweet_df['clean_tweets'].apply(lambda x: re.sub('\n', '', x))

tweet_df['clean_tweets'] =\
tweet_df['clean_tweets'].apply(lambda x: re.sub(r'[^\w\s]', '', x))

```

### 4. Remove amperstand (&)


```python
tweet_df['clean_tweets'] =\
tweet_df['clean_tweets'].apply(lambda x: re.sub('&amp;', '', x))

tweet_df['clean_tweets'] =\
tweet_df['clean_tweets'].apply(lambda x: re.sub('&amp', '', x))
```

### 5. Tokenize, Remove Stopwords, rejoint into string


```python
tweet_df['clean_tweets'] = tweet_df['clean_tweets'].apply(lambda x: tokenize(x))

tweet_df['clean_tweets'] = tweet_df['clean_tweets'].apply(lambda x :\
                                                          remove_stopwords(x))

tweet_df['clean_tweets'] = tweet_df['clean_tweets'].apply(lambda x: " ".join(x))
```

## Use Dask to parallelize the lemmatization of the words.

The goal of lemmatization is to remove the inflection from the words. Returning only the base word.  

Processing each of the 1.3 million tweets one at a time will take a long time becasue lemmatizing a sentence is computationally expensive. To speed up this process we will use the "Dask" package.  

Using Dask we can break the dataframe up in to separate partitions and have each of them processed by a separate core of the processor. This is known as parallel computing.

We begin by getting the number of cores within the computers processor.


```python

parts = os.cpu_count()
parts
```




    12



Then we use Dask to break the Pandas Dataframe up in to the same number of paritions as we have cores. Then we map the 'clean_text' function to each parition and process.  

On my machine a 60 minute operation was reduced to 15 minutes.


```python
import dask.dataframe as ddf
from dask.diagnostics import ProgressBar

dask_df = ddf.from_pandas(tweet_df, npartitions = parts)
result = dask_df.map_partitions(clean_text, meta = tweet_df)
with ProgressBar():
    df = result.compute(scheduler='processes')
```

    [                                        ] | 0% Completed | 16min 52.9sdone
    [###                                     ] | 8% Completed | 17min  1.8sdone
    [######                                  ] | 16% Completed | 17min 10.5sdone
    [##########                              ] | 25% Completed | 17min 18.2sdone
    [#############                           ] | 33% Completed | 17min 24.2sdone
    [################                        ] | 41% Completed | 17min 27.4sdone
    [####################                    ] | 50% Completed | 17min 32.4sdone
    [#######################                 ] | 58% Completed | 17min 36.4sdone
    [#######################                 ] | 58% Completed | 17min 38.2sdone
    [##############################          ] | 75% Completed | 17min 41.0sdone
    [##############################          ] | 75% Completed | 17min 42.3sdone
    [##############################          ] | 75% Completed | 17min 42.8sdone
    [########################################] | 100% Completed | 17min 44.1s


The result is a new dataframe that contains all of the original data plus a new column that contains the lemmatized thext.  

Lemmatizing the text will make it easier to get correct word counts and such.


```python
df.sample(20)

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
      <th>clean_tweets</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1099733</th>
      <td>Great to chat with some of my #TX22 bosses, th...</td>
      <td>C</td>
      <td>great chat tx22 boss robison family town sprin...</td>
    </tr>
    <tr>
      <th>31218</th>
      <td>Staff participated in National Service Day pro...</td>
      <td>L</td>
      <td>staff participate national service day program...</td>
    </tr>
    <tr>
      <th>368405</th>
      <td>When I was at #ParamountHighSchool’s Senior Aw...</td>
      <td>L</td>
      <td>when -PRON- paramounthighschool senior awards ...</td>
    </tr>
    <tr>
      <th>304583</th>
      <td>Wisconsin has lagged in business start-up acti...</td>
      <td>L</td>
      <td>wisconsin lag business startup activity -PRON-...</td>
    </tr>
    <tr>
      <th>484127</th>
      <td>Enjoyed learning about the @wvuLibraries archi...</td>
      <td>C</td>
      <td>enjoy learn archive process even get chance ch...</td>
    </tr>
    <tr>
      <th>954665</th>
      <td>RT @neilwymt: The #SOARSummit is wrapping up, ...</td>
      <td>C</td>
      <td>the soarsummit wrapping coverage continue spec...</td>
    </tr>
    <tr>
      <th>1316896</th>
      <td>RT @CCSTorg: This morning @RepGaramendi met wi...</td>
      <td>L</td>
      <td>this morning meet alumnus share pride help cre...</td>
    </tr>
    <tr>
      <th>952198</th>
      <td>RT @DarrellIssa: RT @GOPoversight: Contempt re...</td>
      <td>C</td>
      <td>contempt resolution vote tally 255 yea 67 nay ...</td>
    </tr>
    <tr>
      <th>1185124</th>
      <td>Interesting and personal story about our SOS n...</td>
      <td>C</td>
      <td>interesting personal story sos nominee everyon...</td>
    </tr>
    <tr>
      <th>506776</th>
      <td>Adam Jobbers-Miller was a patriot, dedicated t...</td>
      <td>L</td>
      <td>adam jobbersmiller patriot dedicated community...</td>
    </tr>
    <tr>
      <th>141123</th>
      <td>.@realDonaldTrump needs to realize: No one is ...</td>
      <td>L</td>
      <td>need realize no one right call question legiti...</td>
    </tr>
    <tr>
      <th>325166</th>
      <td>American workers don’t need NAFTA with a new n...</td>
      <td>L</td>
      <td>american worker do not need nafta new name the...</td>
    </tr>
    <tr>
      <th>1224738</th>
      <td>I’m calling on @realDonaldTrump to sign this i...</td>
      <td>C</td>
      <td>-PRON- be call sign important legislation prom...</td>
    </tr>
    <tr>
      <th>1148018</th>
      <td>Trump Org to Congress: the Constitution degrad...</td>
      <td>L</td>
      <td>trump org congress constitution degrade custom...</td>
    </tr>
    <tr>
      <th>907392</th>
      <td>Want to join Team Moulton? Now accepting appli...</td>
      <td>L</td>
      <td>want join team moulton now accept application ...</td>
    </tr>
    <tr>
      <th>30959</th>
      <td>5 years ago I watched Pres. Obama sign the Aff...</td>
      <td>L</td>
      <td>5 year ago -PRON- watch pre obama sign afforda...</td>
    </tr>
    <tr>
      <th>968455</th>
      <td>Too many students today attend school in crumb...</td>
      <td>L</td>
      <td>too many student today attend school crumble b...</td>
    </tr>
    <tr>
      <th>274732</th>
      <td>RT @mike_pence: Thanks to today's vote in Cong...</td>
      <td>C</td>
      <td>thank todays vote congress one step close repe...</td>
    </tr>
    <tr>
      <th>1221963</th>
      <td>For over 230 years, the U.S. Constitution has ...</td>
      <td>C</td>
      <td>for 230 year us constitution promote value ind...</td>
    </tr>
    <tr>
      <th>1283486</th>
      <td>RT @Jim_Jordan: This isn’t impeachment. This i...</td>
      <td>C</td>
      <td>this be not impeachment this political campaig...</td>
    </tr>
  </tbody>
</table>
</div>




```python

with open('outdata/tweets_clean_df.pkl', 'wb') as f:
    pickle.dump(df, f)

s3.meta.client.upload_file('outdata/tweets_clean_df.pkl',
                           bucket_name,
                           'tweets_clean_df.pkl')

os.remove('outdata/tweets_clean_df.pkl')
```


```python

```


```python

```
