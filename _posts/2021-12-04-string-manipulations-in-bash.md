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
image:               /assets/bg-pawel-czerwinski-ScYk6KKEPUI-unsplash.jpg
cover:               /assets/bg-pawel-czerwinski-ScYk6KKEPUI-unsplash.jpg
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

* How to declare a variable?
* Some general concepts
* Some specific concepts to dig deeper
* How to go further from this article

Now, let's get started!

## Declaring Variables

**Declare a single line variable.** To declare a single line variable, just
declare a variable, followed by an equal sign (`=`) for the assignment, and ends
with the value of the variable, either using a string directly, or using a
command inside a subshell (`$(...)`):

```sh
content="Hello Bash"
creation_date=$(date +"%Y-%m-%d") # 2012-12-04
```

**Declare a multi-line text.** A heredoc is a special-purpose code block that
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

## Substring Removal

Removing a substring can be done inside a shell parameter expansion `${param}`.
Using character `#` can delete a prefix of the string and using character `%`
can delete a suffix of the string. Using the same characters once or twice will
delete the shortest and the longer match respectively. To better remember this,
look at your ISO layout keyboard:

`# 3`, `$ 4`, `% 5`

In a standard keyboard layout, keys 3/4/5 represent `#`/`$`/`%`. Since `#` is
before `$` and `%` is after `$`, deletion with `#` happens from front of string
(`$`), and deletion with `%` happens from end of string (`$`). Here are some
concrete examples:

Deleting the shortest match from front of string:

```sh
d=2021-12-04
echo ${d#*-}  # 12-04
```

Deleting the longest match from front of string:

```sh
d=2021-12-04
echo ${d##*-}  # 04
```

Deleting the shortest match from back of string:

```sh
d=2021-12-04
echo ${d%-*}  # 2021-12
```

Deleting the longest match from back of string:

```sh
d=2021-12-04
echo ${d%%-*}  # 2021
```

## Section 3

## Going Further

How to go further from here?

## Conclusion

What did we talk in this article? Take notes from introduction again.
Interested to know more? You can subscribe to [the feed of my blog](/feed.xml), follow me
on [Twitter](https://twitter.com/mincong_h) or
[GitHub](https://github.com/mincong-h/). Hope you enjoy this article, see you the next time!

## References

- Kewei Shang & Mincong Huang, ["Bash | Tech Resources"](https://github.com/keweishang/tech-resources/blob/master/tool/bash.md), _GitHub_, 2020.
- GNU, ["3.5.3 Shell Parameter
  Expansion"](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html),
_GNU_, 2021.
