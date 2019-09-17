---
layout: post
title: Departure Vision
date: 2019-08-01
description: Overview of the Departure Vision Project
img: penn-station.jpg 
fig-caption: Madness under Penn Station when tracks are listed &copy;Nicolas Kijak
tags: [Data, Python, Serverless, OpenFaaS, Kafka, ML, Mineo]
---
Anyone who's commuted from New York's Penn Station (NYPS) knows how stressful the cattlecall that is a track listing can be.  For those that haven't, the track your train comes in on isn't listed until it has arrived. Officially this is stated as "Tracks will be posted 10 minutes to boarding". Unofficially, especially during rush hour, this is could be minutes before the scheduled departure.

According to [wikipedia as of 2019](https://en.wikipedia.org/wiki/Pennsylvania_Station_(New_York_City)) the station services up to 600,000 passengers per weekday.  New Jersey Transit routinely runs (if you're lucky) "double decker" cars. Imagine a train car, now imagine stacking two of those like bunk beds and fill the walk ways with people.  This gives you an idea what a rush hour train out of NYPS is like on the Northeast Corridor Line.  Add 5 more train lines serving New Jersey alone and you begin to imagine how many people are waiting around for track listings.  Its difficult enough to get through the crowds when your train is called, it's near impossible to get there when your track is on the other side and they announce a train on you.  The wave of commuters you need to wade through is almost too much.  It would be much less stressful if you knew what track to stand by.

### Motivation
One tool seasoned NJ Transit riders know about is the [NYPS Departure Vision](http://dv.njtransit.com/mobile/tid-mobile.aspx?sid=NY) web page.  This is available for all (most?) NJ Transit stations and the same display you find in the station listing tracks.  A quick glance shows that for NYPS there are Amtrak listings as well, this is because NJT uses the lower numbered tracks, Long Island Railroad the upper and Amtrak uses whatever they want. It is their station and tracks after all. But the information on this is limited to real time track numbers and train statuses.

This got me thinking about trying to make a track predictor.  The track information is available. The problem was getting that track data as a time series. NJ Transit doesn't have an API for this information so that meant a whole engineering problem to solve.  It also allowed me to experment with some tech I haven't used and would lead into some Machine Learning, or at the very least a rudimentary statistics lesson.  Additionally, once I could pull data down for one station I could expand to any station serviced by departure vision and use that to mine additional information like cancelation rates or average delays.

## Technical Overview
### Getting Track Data
The first issue was getting track information as it changed.  Since there's no official feed I would have to periodically poll Departure Vision and scrape the info. Sounds like an easy cronjob.  That's too boring though. So I took some RaspberryPis I had, created a [Docker Swarm](https://docs.docker.com/engine/swarm/) and installed [OpenFaaS](https://www.openfaas.com/), which is a open source serverless platform.

### Storing Track Data
I used this project to experment with serverless function design.  I determined to make the functions as simple and single purpose as possible. I wanted raw storage to be cloud like and found [Minio](https://min.io/index.html), a prive S3 API compatable private object store. The first function simply grabing the specified station's departure vision board and storing in Minio as HTML. 

### Creating a Data Stream
I wanted to learn Kafka. I have used AWS Kinesis, which is similar in design, but Kafka seems to have the larger ecosystem with more interesting tools and support.   I stood up a single node Kafka broker with a topic which would be the data from departure vision as it came in.  

Minio has multiple hooks for event call backs, much like S3.  So I created a new function that would get the event, fetch the html from S3 convert it to JSON and publish on Kafka.

### Creating a Change Feed
Being in the Kafka ecosystem mean I wanted to learn what was available to process a stream.  Kafka has a [framework](https://kafka.apache.org/documentation/streams/) for processing data with design goals similar to [Spark Streaming](https://spark.apache.org/streaming/) and [Apache Beam](https://beam.apache.org/).  As this is a stateful application due to having to know the previous state to determine changes, I needed to use the lower level Processor API. After figuring out the higherlevel Streams DSL wouldn't help me it was quite easy to create a processor to read a stream and produce the change stream.  In this case when a train appears on the board, when it departs and drops off the board and any in between changes like track notices or delays.

### Predictors
With the above streams and raw data store. I can move on to the final step of creating a predictor.  I plan on having a simple stat counter but also want to take the opertunity to drill into more advanced analysis with Scikit-Learn and eventually over do it with some Tensorflow.  The final learning element here is the pipeline training the model and finally hosting it as a function in OpenFaaS.

## Further Reading
- [Code](https://github.com/nkijak/departure-vision-recorder)
- In Depth Posts TBA
