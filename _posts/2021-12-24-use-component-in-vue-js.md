---
layout:              post
type:                Q&A
title:               Using Component in Vue.js
subtitle:            >
    Avoid having thousands of lines of code in a single file.

lang:                en
date:                2021-12-24 08:05:27 +0100
categories:          [frontend]
tags:                [javascript, vuejs, vuejs2]
ads_tags:            []
comments:            true
excerpt:             >
    TODO

image:               /assets/vuebigwhite.png
cover:               /assets/vuebigwhite.png
article_header:
  type:              overlay
  theme:             dark
  background_color:  "#203028"
  background_image:
    gradient:        "linear-gradient(135deg, rgba(0, 0, 0, .6), rgba(0, 0, 0, .4))"
wechat:              false
---

## Question

My `*.vue` files are getting bigger and bigger. It's hard to maintain. I would
like to split the file and move part of the structure (HTML) and the logic
(JS) to another location. Is it possible?

## Answer

You can do that by using components. You can extract part of the structure from
the original vue file to a new file, and then import the component again into
the original one. Let's say we have two files:

* `my-page.vue` -- the main page that contains most of the logic and
  it's getting big.
* `my-component.vue` -- the new component that you are creating

First of all, you need to declare the structure in the component file.

```html
<template>
  <div>
    <!-- TODO Add content for your resource (component) here -->
  </div>
</template>
<script>
export default {
  props: {
    resourceId: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      isLoading: true,
      resource: null
    }
  },
  methods: {
    loadResource() {
       // TODO Implement async HTTP call here
    }
  }
}
</script>
```

Provide some text to explain what is it, requirements, how does it work, etc.

```java
// Then follow by a code snippet to provide a concrete example
```

Explain the example a bit or continue the answer here.

## Going Further

How to go further from here?

- Other answers that we didn't see
- More context about the framework
- Other tricks regarding the question

Hope you enjoy this article, see you the next time!

## References
