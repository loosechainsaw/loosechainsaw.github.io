---
layout:     post
title:      C++ Singletons
date:       2020-02-08 22:19:00
author:     Blair Davidson
summary:    C++ Singleton Design Pattern
categories: C++
thumbnail:  heart
tags:
 - C++
 - Singleton
 - Design Patterns
---
The singleton design pattern is one of the original patterns for the mighty design patterns book, and also one of the
most commonly used design patterns in the wild. The purpose of the Singleton pattern is to enforce that there can only
be one instance of a particular type T. While this sounds simple enough, in practice there are various traps, especially
in C++. We will present the Singleton in Modern C++ and show all the various traps along the way.

So without further ado here it is:

{% highlight cpp %}

#include <iostream>
#include <mutex>

class singleton {
public:
    static singleton* instance();
    singleton(singleton const&) = delete;
    singleton& operator=(singleton const&) = delete;
    singleton(singleton&&) = delete;
    singleton& operator=(singleton&&) = delete;
    ~singleton();
private:
    singleton();
    int state_ = 1;

    static singleton *instance_;
    static std::once_flag flag_;
};

singleton *singleton::instance_ = nullptr;
std::once_flag singleton::flag_;

singleton *singleton::instance() {
    std::call_once(flag_, []{
        instance_ = new singleton;
    });
    return instance_;
}

singleton::singleton() {
    std::cout << "singleton()" << std::endl;
}

singleton::~singleton() {
    std::cout << "~singleton()" << std::endl;
}

int main() {

    auto *inst = singleton::instance();

    return 0;
}

{% endhighlight %}
`
So lets look at some of the obvious design decisions made to the Singleton class. Firstly all of the copy and move
constructors are marked as deleted. This is fairly obvious as a we want to have a single instance only. The constructor
is also made private, but why? 
