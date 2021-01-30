---
title: Automating the retrieval of tweets about Cafeyn, Blendle and MiLibris
updated: 2021-01-30 11:32
category: data
tags: Twitter, API, Data, Cafeyn, Blendle, Milibris, Python, R, Raspberry Pi
---

If you're into data viz and you're active on Twitter, you can find some incredible content by following the right people. Within the top submissions of [#TidyTuesday](https://twitter.com/hashtag/TidyTuesday?src=hashtag_click) you will realize that a few people tend to consistently deliver some very high quality work. 
I've asked myself what led to them to develop such skills, but even more importantly, what motivated them to continuously take part in the challenge on a weekly basis.
Clearly, there was a personal affinity with taking part in the challenge, as well as the content they could work on.

## Inspiration for a personal project
It's been a while that I wanted to work on a personal project. The work of [Nicolas Kayser-Bril](https://twitter.com/nicolaskb) has been particularly inspiring. In particular, he built a Twitter bot called ["Context for my weather"](https://twitter.com/weathercontext) that a user could tweet to with a specific city name. The bot would then dynamically create a visualization of the historical weather data for that location. Here is an example of the reply tweet you would get from the bot if you tweet at him with the city "Amsterdam".

![alt text]({{site.url}}/assets/context_for_my_weather_amsterdam.jpg)


## Twitter discourse
Since I'm an active Twitter user, I thought I could start there as well. As a former communication graduate, I am particularly interested in analyzing online discussions as a measure of public sentiment about certain topics, people or brands. The field I work in - online news aggregators - is an interesting subject. How does the public's perception of main news aggregators evolve across time? Do different cultures react differently to these new players that are changing the traditional landscape of journalism? How can key events such as a feature launch or a new market expansion can shape the way audiences perceive the brands? Those are some of the questions that we can attempt to understand better by collecting Twitter data from users tweeting about those comapnies. 

### Collecting the data
The first step to get there is to collect data. Dozens of tweets are sent every day about Cafeyn, Blendle and Milibris. I needed a script that would search all recent Tweets related to one of those three companies once a day. The Twitter API supports such basic requests as long as they do not exceed certain rate limits and do not go back more than 30 days. That means that historical data is very limited, at least on a free plan. 

The script needs to run once a day and setting a reminder to run it is clearly not efficient. It needs to be scheduled as a Cron job on a server. I don't have a server but I received a Raspberry pi for my birthday, which can definitely be used as a connected device that handles such scheduled tasks. Clearly, I will need to find more interesting ways to use my pi but for now, this will do.  

So for now, I have a [Github repository of my project](https://github.com/arno12/twitter) containing the needed environment, and the current scripts I use to retrieve the data. It first makes sure that the project state is up to date with master, which I might modify from my regular laptop when working on the R script, and then runs the main job to retrieve the latest tweets and add them incrementally to a csv. Ideally, I'd like to improve this process by not relying on a csv file that I commit to Github but rather use some server storage like Amazon S3 or Google Cloud Storage. But with less than 2000 rows of data, I have other things to prioritize.

### Transforming the data
The most important learning as a data analyst is that data is never clean, or at least never clean enough for what you intend to do with it. Luckily, that step of the process has become very intuitive over the years, and using the Tidyverse ecosystem in R, it's a matter of minutes to transform a dataframe into the shape needed for explorative analysis. 

### Generating some first insights
The first two processes could still be improved but looking back at all the ideas I had that never materialized, I came to the conclusion that if you want your personal projects to happen, you need to get something done, working on a minimal, still flawed example before iterating over it to improve its individual parts.

I decided to transform some of the descriptive insights from my data as a first step, which enabled me to look at two thing in particular.

1. **Overview of the tweets collected over time**<br/><br/>
This graph shows how many tweets we retrieve from each company across time.  

![alt text]({{site.url}}/assets/2021-01-30-14h41_tweets_over_time.png)

As of today (30/01/2021), I collected

* Blendle: 1,031 tweets
* Cafeyn: 658 tweets
* Milibris: 45 tweets

These include a lot of tweets from the companies' own Twitter accounts, which I exclude for other analyses but we could use them to analyze our own tone of voice on Twitter over time.

The reason why you see gaps is because I didn't have the script working on a daily basis until beginning of January. 
From then on, the Raspberry Pi started running it once a day via a Cronjob.

2. **Overview of the most influential accounts tweeting about each company**<br/><br/>
This graph shows accounts that were more influential (likes and retweets) from the data we have collected. Since there are multiple languages at hand, there is a color code related to those accounts, meaning that a single accounts tweeting in n languages could results in that account appearing n times. Some accounts with a single tweet without engagement have been filtered out. Note that the scales are logs to contain outliers. 
![alt text]({{site.url}}/assets/2021-01-30-14h40_tweet_users.png)

I'd like to find a way to make this graph more interactive so that a user could click on an account's name and drill down the tweets that are behind it. 

More graphs will be added along the way :) A wordcloud and a sentiment analysis per language/company are obvious next steps. More advanced NLP insights will come once there is more data available.

## Next steps
There are still a lot of things to be improved. Here are a few I wrote down

1. Have a setup where the tweets are gathered, processed and outputed on a page from my website every day (kind of an interative graph that is automatically refreshed every day). The main challenge is that the pi would need to run and render the r scripts itself, or if it's a dynamic graph it would need to be some kind of application (using R Shiny is an option)

2. Extend Twitter API calls to a wider spectrum than just Twitter search results. There is definitely more interesting data to be collected.

3. Expand analysis onto NLP when there is more content. The challenge will be handling different languages. 

If you have feedback, feel free to let me know on Twitter [@NosyOwl](https://twitter.com/NosyOwl)