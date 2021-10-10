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
This article is written with Temporal Go SDK v1.10.0 (15 Sept 2021).

After reading this article, you will understand:

* The difference between retryable and non-retryable error at acvitity level
* How to write unit tests?


If you don't have time to read the entire article, here is a table for
summarizing the difference.

Scope    | Error Type       | Methods | Retryable (Default) | Retryable (Override)
:------: | :--------------- | :------ |:------------------- | :-------------------
Activity | `ApplicationError` | `temporal.NewNonRetryableApplicationError()` | No | -
Activity | `ApplicationError` | `temporal.NewApplicationError()` | Yes | -
Activity | Other errors | `fmt.Errorf()`, `errors.New()` | Yes | Retry Policy

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

`ApplicationError` determines whether an error is retryable using its internal
boolean attribute `nonRetryable`:

```go
// go.temporal.io/sdk@v1.10.0/internal/error.go
type (
	// ApplicationError returned from activity implementations with message and optional details.
	ApplicationError struct {
		temporalError
		msg          string
		errType      string
		nonRetryable bool
		cause        error
		details      converter.EncodedValues
	}
	...
}
```

One possible usecase for `temporal.NewNonRetryableApplicationError(...)` is when
interacting with a third-party service. When that service returns a
deterministic error indicating that required action cannot be performed, you may
not want to retry. For example, when a resource deletion request is rejected by
the third party service because it is still in used, you probably
don't want to retry. Therefore, calling
`temporal.NewNonRetryableApplicationError(...)` is a good choice.

## Non-Retryable Error Types in Retry Policy

Another way to define non-retryable error types for activity is to provide a
custom `RetryPolicy` as part of the `ActivityOptions` or `ChildWorkflowOptions`.
For example, to avoid retrying errors of type `MyError`, we can make it as
non-retryable as follows:

```go
func MyWorkflowWithRetryPolicy(ctx workflow.Context, name string) (string, error) {
	ctx = workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
		StartToCloseTimeout: 10 * time.Second,
		RetryPolicy: &temporal.RetryPolicy{
			InitialInterval:    1 * time.Second,
			BackoffCoefficient: 2,
			MaximumInterval:    1 * time.Minute,
			MaximumAttempts:    5,
			NonRetryableErrorTypes: []string{"MyError"}, // HERE
		},
	})
	...
}
```

The reason why Temporal has retry policy are probably because:

* `temporal.NewNonRetryableApplicationError(...)` does not fit all the usecases.
  Sometime users already know the error types that they don't want to retry, but
  they don't want to determine the error types themselves for each activity and
  fire a non-retryable applicaton error, since it makes the activity verbose.
* Bringing the control at workflow level. An activity can be used for multiple
  workflows, e.g. primitive activities for GitHub, Slack, Build, Kubernetes, etc.
  Depending on the case of each workflow, some may want to retry while others
  don't.
* Actually retry policy is not only used to activity. It can be used as part of
  the activity options, child workflow options, or event the top-level workflow
  options.

## TODO

go doc

https://github.com/temporalio/temporal/blob/06b863741d386fb540420431bf157dd26509c464/service/history/workflow/retry.go#L110-L141

## Conclusion

You can subscribe to the [feed of my blog](/feed.xml), follow me on [Twitter](https://twitter.com/mincong_h) or [GitHub](https://github.com/mincong-h/). Hope you enjoy this article, see you the next time!
