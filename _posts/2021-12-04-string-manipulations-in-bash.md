---
layout:              post
title:               String Manipulations in Bash
subtitle:            >
    Given one sentence to expand the title or explain why this article may interest your readers.

lang:                en
date:                2021-12-04 12:10:41 +0100
categories:          [bash]
tags:                [bash, scripting]
ads_tags:            []
comments:            true
excerpt:             >
    TODO
image:               /assets/bg-coffee-84624_1280.jpg
cover:               /assets/bg-coffee-84624_1280.jpg
article_header:
  type:              overlay
  theme:             dark
  background_color:  "#203028"
  background_image:
    gradient:        "linear-gradient(135deg, rgba(0, 0, 0, .6), rgba(0, 0, 0, .4))"
wechat:              false
---

## Introduction

When working in the software industry, no matter you are a software engineer, a
data scientist, a support enginner, or any other roles, you probably need to
know some basic skills about Bash to
improve your productvity. This can be useful for automating complex tasks in
your terminal, generating file for configuration or documentation, sharing a
command with your teammates, etc. In this post, we are going to explore some
frequently used techniques about string manipulations.

After reading this article, you will understand:

* Some prequisites
* Some general concepts
* Some specific concepts to dig deeper
* How to go further from this article

Now, let's get started!

## Declaring Variables

**Declare a multi-line text.** A heredoc is a special-prupose code block that
tells the shell to read input from the current source until it encounters a line
containing a delimiter. EOF (end of file) is a commonly used delimiter but it's
not mandatory. You can use JSON, YAML, TEXT, or any other delimiter that you
think relevant for your situation. The syntax for Heredoc in Bash is:

```sh
COMMAND << DELIMITER
Here is the long description
...
DELIMITER
```

Here is an example for generating a YAML file, where we generate the heredoc and
print it inside a subshell
and assign the result to a variable:

```sh
content=$(cat << YAML
cluster:
  type: $TYPE
  name: elasticsearch-$TYPE
  date: $(date +"%Y-%m-%d")
YAML
)
```

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
