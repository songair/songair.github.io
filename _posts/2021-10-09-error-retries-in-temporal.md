---
layout:              post
title:               Error Retries in Temporal Workflow
subtitle:            >
    Retry or not retry?

lang:                en
date:                2021-10-09 22:08:20 +0200
categories:          [temporal]
tags:                [temporal, go]
comments:            true
excerpt:             >
    TODO

image:               /assets/bg-pablo-garcia-saldana-lPQIndZz8Mo-unsplash.jpg
cover:               /assets/bg-pablo-garcia-saldana-lPQIndZz8Mo-unsplash.jpg
article_header:
  type:              overlay
  theme:             dark
  background_color:  "#203028"
  background_image:
    gradient:        "linear-gradient(135deg, rgba(0, 0, 0, .6), rgba(0, 0, 0, .4))"
wechat:              false
ads:                 none
---

## Introduction

When working with Temporal to build workflows, you will have to face to error
handling at some point because workflow and activity can fail for different
reasons. Temporal Go SDK defines its [error
handling](https://docs.temporal.io/docs/go/error-handling/) logic and [activity
and workflow retries](https://docs.temporal.io/docs/go/retries/) in the official
documentation. But whenever I visit those pages, I always feel that
it's missing something that I need as a developer. So I decide to write this
article, to share my understanding of error retries in Temporal in Go SDK, as a
complementary to the official documents. And hopefully, it will clarify
different situations and give you a clearer picture of how errors are retried.

After reading this article, you will understand:

* The difference between retryable and non-retryable error at acvitity level
* How to write unit tests?


If you don't have time to read the entire article, here is a table for
summarizing the difference.

## Retryable and Non-Retryable Application Error

By default, Temporal retries activities, but not workflows. According to
official documentation [Error Handling in
Go](https://docs.temporal.io/docs/go/error-handling/), if the activity returns
an error as `errors.New()` or `fmt.Errorf()`, that error will be converted to
`*temporal.ApplicationError`, which is retryable. If you don't want the error to
be retried, you can return a non-retryable application error from the activity.

```go
func MyActivity(ctx context.Context, name string) (string, error) {
	...
	// retryable
	return "", fmt.Errorf("oops")
}
```

```go
func MyActivity(ctx context.Context, name string) (string, error) {
	...
	// retryable
	return "", temporal.NewApplicationError("oops", "test")
}
```

```go
func MyActivity(ctx context.Context, name string) (string, error) {
	...
	// non-retryable
	return "", temporal.NewNonRetryableApplicationError("oops", "test", err)
}
```

This is easy to understand: Temporal wants to provide a fault-tolerant system so
that it can retry automatically when thing goes wrong. So at activity-level,
error are retried, unless user asks Temporal to not retry explicitly via wrapper
method `temporal.NewNonRetryableApplicationError(...)`.

## TODO
go doc

https://github.com/temporalio/temporal/blob/06b863741d386fb540420431bf157dd26509c464/service/history/workflow/retry.go#L110-L141

## Conclusion

You can subscribe to the [feed of my blog](/feed.xml), follow me on [Twitter](https://twitter.com/mincong_h) or [GitHub](https://github.com/mincong-h/). Hope you enjoy this article, see you the next time!
