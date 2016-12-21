+++
title = "Unravelling Logs - Part 1"
description = "Making sense of logs in distributed systems"
date = "2015-11-10T08:06:37Z"

+++

Traditionally in enterprise IT environments, logs have tended to be used mainly for auditability purposes, and to enable post incident root cause analysis. 

Log collection is centralised if you’re lucky, and that in itself is regarded as a fairly major technical achievement. 

This kind of works OK when systems are mainly stand alone, and problems can be diagnosed from looking at an individual machine. 

Even in those situations though, logs tend to be the second point of call after an incident has already occurred and you’ve been notified about some failure from your service monitoring system – I’m sure many of us are familiar with the situation of investigating an outage, only to find that the machine involved had been logging partial failures for a long time without anyone noticing.

When you’re trying to manage large scale distributed systems, logs present a very different kind of problem, and a different kind of opportunity. 

For a start, there’s a vast amount of data, far too much for any sane person to trawl through, and secondly, for any particular point in time to make any sense you potentially need to correlate log entries across a large number of services and machines. 

Log data also starts to become much more important, as it gives you access to the real time information about the system state and often holds information that service monitoring won’t necessarily tell you, about events that may be transient  or trending but have impact on stuff like performance for example. 

At DataCentred we’ve started seeing logs as the heartbeat of our platform, and being able to sort and present that data in useful ways and also act on it automatically is one of the key development focuses for our team.

In order to develop ways of making our logs more useful, there are a couple of key things we need to do. 

Firstly we need a reliable mechanism for gathering and centralising log data, and that data then needs to be structured so we can do search and analysis. 

In this context structuring means splitting up into it’s consitituent parts such as timestamp, log message, creating process etc. and storing it in some kind of database format. 

This is obviously totally dependent on understanding the format of the log files, and this in itself can be massively variable – standards in this space are fairly fluid and by no means widespread so the format of log files varies tremendously.

Whilst there are proprietary solutions out there which provide functionality which addresses some of these issues, like Splunk for example, these can be very complex, and are often extremely expensive. 

Luckily for us, the open source world has a set of fantastic tools for doing exactly what we need – the ELK stack of ElasticSearch, Logstash and Kibana.

[ElasticSearch](https://www.elastic.co/products/elasticsearch) is a distributed search engine, written in Java and built on top of the [Apache Lucene](https://lucene.apache.org/core/) search technology. It’s massively horizontally scalable and super fast. 

[Kibana](https://www.elastic.co/products/kibana) provides a highly configurable graphical frontend into ElasticSearch, and last but not least [Logstash](https://www.elastic.co/products/logstash) is a log storage and processing system.

Over this short series of blog posts, I’ll be going end to end across some of the approaches we’re adopting, including how we’re using the ELK stack, which hopefully may be of some use to others out there in starting to unravel your logs.
