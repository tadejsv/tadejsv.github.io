---
layout: post
title: "Katex test"
date: 2020-08-07 12:00:00
tags: test katex
---

> Just testing Katex here and code here.

<!--more-->

# Katex

Here's a Katex equation:
{% katex display %}
c = \pm\sqrt{a^2 + b^2}
{% endkatex %}

# Code

Here's a code example:

{% highlight python %}

# Configuration is wrapped in one object for easy tracking and passing.
class RNNConfig(): 
    input_size=1
    num_steps=30
    lstm_size=128
    num_layers=1

config = RNNConfig() 
{% endhighlight %}