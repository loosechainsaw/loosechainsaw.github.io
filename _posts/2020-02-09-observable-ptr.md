---
layout:     post
title:      Safe Non Owning Pointers
date:       2020-02-09 22:19:00
author:     Blair Davidson
summary:    Modeling pointers in C++ that are non owning semantics in an explicit manner
categories: C++
thumbnail:  heart
tags:
 - C++
 - pointers
 - non owning
---

Non Owning C++ Pointer Types

One of the biggest issues that arise in C++ is who owns a particular pointer. For example
consider the simple function declaration below.

{% highlight cpp %}
int* function();
{% endhighlight %}

