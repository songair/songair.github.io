---
layout:              post
title:               Audit Logs
subtitle:            >
    Implementing a simple audit logs solution with Java JAX-RS.

lang:                en
date:                2021-11-20 08:40:44 +0100
categories:          [java-core]
tags:                [java, jetty, jersey, api, jax-rs]
ads_tags:            []
comments:            true
excerpt:             >
    TODO
image:               /assets/bg-markus-spiske-gnhxvdGmGG8-unsplash.jpg
cover:               /assets/bg-markus-spiske-gnhxvdGmGG8-unsplash.jpg
article_header:
  type:              overlay
  theme:             dark
  background_color:  "#203028"
  background_image:
    gradient:        "linear-gradient(135deg, rgba(0, 0, 0, .6), rgba(0, 0, 0, .4))"
wechat:              false
---

## Introduction

Today I would like to discuss audit logs with you. Audit logs are logs for
auditing. They are events that keep track of creation, modification, deletion,
or any other operation that mutates the state of a given resource. This resource
can be a database, a pipeline, or anything valuable for the company. You may
want to keep track of these events since they can be useful for security
analysis, troubleshooting, compliance, auditing, keeping track of the lifecycle
of a data-store, etc, depending on your role. During
my work at Datadog, I had the chance to implement a simple audit solution for an
internal tool. That's why I want to write down some thoughts and hopefully, they
will be useful for you as well.

After reading this article, you will understand:

* Requirements for audit logs
* Principles when implementing audit logs
* Focus deeper in Java solution using JAX-RS
* How to go further from this article

Now, let's get started!

## Requirements For Audit Logs

Generally speaking, there are some information that we care about:

- **Resource.** We want to know what is being accessed or modified. Therefore,
  we may want to record the resource ID, resource name, resource type, resource
  group, or any other information related to this resource. Regarding to RESTful
  API, the resource ID can be the path, which is usually the representation of
  the resource.
- **Time.** We want to know when does this happen precisely. This is important
  to construct a timeline for a bigger event, like an incident, an attack, or
  the lifecycle of a resource.
- **Action.** We want to know what is being done on that resource. It provides
  an accurate description about the type of operation. Some typical examples are
  "create", "read", "delete", "update", etc.
- **User.** We want to know "who did that?" so that we can find out more
  information based on that user or better understand the motiviation of this
  operation. The user information may contain: the first name, last name, email,
  department, organization unit, employee ID, etc.

We can eventually go further by adding more metadata to faciliate the search,
making the description more human readable, etc. But I believe that these are not
requirements, but enhancements to make the feature more usable.

Then on the business side, there are other also some requirements:

- **Retention.** The retention of the audit logs. We want them to store longer
  than normal logs because they are specific logs for investigation. These are
  precious events helping us to redraw the big picture.
- **Access**. Perhaps not everyone should be accessed to audit logs. Taking
  Datadog's product ["Audit
  Logs"](https://docs.datadoghq.com/fr/account_management/audit_logs/) as an
  example, only administrator or security team member can access Audit Logs. As
  an individual, you can only see a stream of your own actions.

I probably didn't cover everything in the section. If you have other ideas,
please let me know what you think in the comment section below.

## Principles When Implementing Audit Logs

When implementing audit logs, I believe here are the principles to follow and I
will try to explain why.

**Hooking into the lifecycle.** When implementing audit logging, we need to
decide where should we put the code. I believe that the best option is to hook
your logic into the lifecycle of the framework that you use. Then, you will be
able to log before or after an event. For example, if you use Java
Persistence API (JPA), you can implement your logic using `@PrePersist`,
`@PreUpdate`, `@PreRemove` callbacks. Or if you use Java RESTful API
(JAX-RS), you can implement interfaces `ContainerRequestFilter` or
`ContainerResponseFilter` to handle the audit logging, respectively before the
request being handled or after the response being created. By hooking into the
lifecycle, we ensure that the audit logging is decoupled from the actual
business logic. We avoid spamming the codebase by avoiding adding the audit logs
into every method. It also makes it clear about when does the audit actually happen.

**Avoid blocking the actual event.** When adding audit logs, we should also
avoid blocking actual events so that user's action won't be blocked or delayed.
This is because sometime the audit logging requires API calls, which means that
they may be slow or suffer from network issues. So my suggestion is to use
asynchronous implementation so that the actual event will be handled correctly.
As for network issue or other types of error, we can make it fault-tolerent by
adding a retry mechanism. We can also consider using batch API call to group
multiple events together.

## Java Solutions

In this section, I would like to go further into Java to discuss how to
implement a simple audit logging solution for Java RESTful APIs. Here I am going
to list 3 solutions based on Java Servlet, Jetty, and JAX-RS (Jersey).

### Java Servlet

For those who don't know Java Servlet, here is a quick introduction. Java
Servlet or nowadays Jakarta Servlet, is a Java software component that extends
the capabilities of a server. It's commonly used for implementing web containers
for hosting web applications, similar to PHP and ASP.NET. The evolution of Java
Servlet is part of the Java Specification Requests
([JSRs](https://jcp.org/en/jsr/all)). The latest one is Java Servlet 4.0
(JSR-369) started on 2017.

In our case, we can implement a simple Servlet filter to intercept the HTTP
request or response using the `doFilter()` method. Inside the method, you
must call the filter chain to pass the request and response to the next
filter so that they are handled. Otherwise, the request will be dropped
(filtered), which is not desired. Then, you can implement the actual auditing
logic before or after the chain. I perfer after the chain because in this case,
we will have the information of both the HTTP request and the HTTP response,
which makes the auditing logging more complet.

```java
import javax.servlet.*;
import java.io.IOException;

public class SimpleServletFilter implements Filter {

    public void init(FilterConfig config) throws ServletException {}

    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
        // before the request being handled
        chain.doFilter(request, response);
        // after the response being created
    }

    public void destroy() {}
}
```

### Jetty Server

If you are using Jetty as the solution for your Java server, you can extend
`AbstractNCSARequestLog` to provide a custom access log solution in the
pseudo-standard NCSA commong log format. To do that, you can create a request
log handler which handles the request log, and then use the handler in your
Jetty server:

```java
var logHandler = new RequestLogHandler();
logHandler.setRequestLog(new MyRequestLog());
server.setHandler(logHandler);
```

where the implementation of `MyRequestLog` looks like this:

```java
public class MyRequestLog extends AbstractNCSARequestLog {
    public MyRequestLog() {
        // configure options here: timezone, locale, extended, IP address, ...
    }

    @Override
    public void write(String entry) throws IOException {
        logger.info(entry);
    }
}
```

The problem of this approach is that the final result must be a string and it
must looks like an access log. So if you need a more custom solution, then you
will need to find another way to handle it.

## Going Further

How to go further from here?

## Conclusion

What did we talk in this article?
Interested to know more? You can subscribe to [the feed of my blog](/feed.xml), follow me
on [Twitter](https://twitter.com/mincong_h) or
[GitHub](https://github.com/mincong-h/). Hope you enjoy this article, see you the next time!

## References

- ["LDAP Data Interchange Format (LDIF)"](https://en.wikipedia.org/wiki/LDAP_Data_Interchange_Format), _Wikipedia_, 2021.
- ["Representational state
  transfer"](https://en.wikipedia.org/wiki/Representational_state_transfer),
  _Wikipedia_, 2021.
- ["Auditing with JPA, Hibernate and Spring Data
  JPA"](https://www.baeldung.com/database-auditing-jpa), _Baeldung_, 2020.
- ["Chapter 10. Filters and
  Interceptors - Jersey"](https://eclipse-ee4j.github.io/jersey.github.io/documentation/latest/filters-and-interceptors.html),
_Eclipse EE4J_, 2021.
- ["Filter (Java(TM) EE 7 Specification APIs)"](https://docs.oracle.com/javaee/7/api/javax/servlet/Filter.html), 2015.
