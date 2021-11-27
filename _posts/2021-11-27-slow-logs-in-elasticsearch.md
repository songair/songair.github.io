---
layout:              post
title:               Slow Logs In Elasticsearch
subtitle:            >
    How to better understand slow logs in Elasticsearch?

lang:                en
date:                2021-11-27 09:19:02 +0100
categories:          [elasticsearch]
tags:                [elasticsearch]
ads_tags:            []
comments:            true
excerpt:             >
    TODO

image:               /assets/bg-nareeta-martin-cOWn-bYGmlc-unsplash.jpg
cover:               /assets/bg-nareeta-martin-cOWn-bYGmlc-unsplash.jpg
article_header:
  type:              overlay
  theme:             dark
  background_color:  "#203028"
  background_image:
    gradient:        "linear-gradient(135deg, rgba(0, 0, 0, .6), rgba(0, 0, 0, .4))"
wechat:              false
---

## Introduction

If you are using Elasticsearch in production, you are probably familiar with
slow logs. Slow logs are logs provided by Elasticsearch, which help you to
troubleshoot the slowness of the write path (index) or the read path
(search). It's useful for determinating the root cause of the problem and may
provide hints to mitigations and solutions. In this article, I want to discuss
slow logs with you, in particular:

* Some prequisites
* Some general concepts
* Some specific concepts to dig deeper
* How to go further from this article

Now, let's get started!

## Section 1

Let's analyze a slow log provided Elasticsearch official documentation ([link](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-slowlog.html)):

> [2030-08-30T11:59:37,786][WARN ][i.s.s.query              ] [node-0] [index6][0]
> took[78.4micros], took_millis[0], total_hits[0 hits], stats[],
> search_type[QUERY_THEN_FETCH], total_shards[1],
> source[{"query":{"match_all":{"boost":1.0}}}], id[MY_USER_ID],

If we split the log into pieces, we can obtain the following table:

Field | Value | Description
:--- | :--- | :---
Timestamp | 2030-08-30T11:59:37,786 | The timestamp of the log.
Log Level | WARN | It means that the query execution time is higher than the query "warn" threshold. This is configured in index settings API.
Logger | i.s.s.query | This is a concise version of logger name (class name): "index.search.slowlog.query". This is useful for use to determine the type of the slowness: index or search.
Node | node-0 | It means that the slowness happens on node-0. If you select a time range and group by nodes, it will help you the determine the distribution of the slowness cross the cluster. In other words, determine if the slowness happens on one single node or happens on multiple ones. Also, if you name the node with some special meaning, e.g. roles (fresh, warm, cold), you will be able to determine which roles are slow.
Index name | index6 | The name of the index that is impacted. We have similar troubleshooting ideas as "node" mentioned above. Additionally, you may have the customer's id, product type, timestamp, or other useful information as part of the index names. Therefore, it's useful for you to determine which customer, which product, or which range of indices are impacted.
Shard | 0 | The shard id.
Took | 78.4micros | Execution time in humain readable format. It determines how slow the query is.
Took in milliseconds | 0 | Execution time in milliseconds in numeric value. If you have the possibility to transform this value in your log pipelines, you can calculate filter the queries that are slower than certain threshold, e.g. ">= 2 seconds". Therefore, you can reduce noises and better indentify the source of the problem. This transformation can be done using a grok parser. Here is a great beginner's guide about [Logstash Grok](https://logz.io/blog/logstash-grok/) and here is the documentation of [Grok Parser in Datadog](https://docs.datadoghq.com/logs/log_configuration/processors/).

## Section 2

## Section 3

## Going Further

How to go further from here?

## Conclusion

What did we talk in this article? Take notes from introduction again.
Interested to know more? You can subscribe to [the feed of my blog](/feed.xml), follow me
on [Twitter](https://twitter.com/mincong_h) or
[GitHub](https://github.com/mincong-h/). Hope you enjoy this article, see you the next time!

## References

- Ran Ramati, Gedalyah Reback, ["A Beginner’s Guide to Logstash Grok"](https://logz.io/blog/logstash-grok/), _logz.io_, 2020.
- ["Processors \| Datadog Documentation"](https://docs.datadoghq.com/logs/log_configuration/processors/), _docs.datadoghq.com_, 2021.
- Vineeth Mohan, ["Slow Logs in Elasticsearch"](https://qbox.io/blog/slow-logs-in-elasticsearch-search-index-config-example), _qbox.io_, 2018.
