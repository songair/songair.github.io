---
layout:              post
title:               Internal Structure Of Snapshot Repository
subtitle:            >
    Given one sentence to expand the title or explain why this article may interest your readers.

lang:                en
date:                2021-09-04 10:11:10 +0200
categories:          [elasticsearch]
tags:                [elasticsearch, java]
comments:            true
excerpt:             >
    TODO
image:               /assets/bg-dmitrij-paskevic-YjVa-F9P9kk-unsplash.jpg
cover:               /assets/bg-dmitrij-paskevic-YjVa-F9P9kk-unsplash.jpg
article_header:
  type:              overlay
  theme:             dark
  background_color:  "#203028"
  background_image:
    gradient:        "linear-gradient(135deg, rgba(0, 0, 0, .6), rgba(0, 0, 0, .4))"
wechat:              false
ads:                 none
---

This article is translated with Google Translate and reviewed by Mincong.
{:.info}

## Introduction

If you use an Elasticsearch cluster in production, I believe you must have heard of Elasticsearch's [snapshot and restore feature](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/snapshot-restore-apis.html), because it is an important means to ensure that cluster data is not lost. There are a lot of materials on the Internet about how to use Elasticsearch snapshots, but there are very few articles about its implementation. Today, I want to discuss with you the internal structure of the Elasticsearch snapshot repository. Knowing it can give us a better understanding of how Elasticsearch's snapshots work, and can also provide more ideas for troubleshooting when there is a problem in production.

After reading this article, you will understand:

- What is a snapshot repository?
- Different files used by a snapshot repository
- Learning more about `index-N` files
- Learning more about the `index.latest` file
- Learning more about snapshot information files

Without further ado, let's get started right away!

## Conclusion

You can subscribe to the [feed of my blog](/feed.xml), follow me on [Twitter](https://twitter.com/mincong_h) or [GitHub](https://github.com/mincong-h/). Hope you enjoy this article, see you the next time!
