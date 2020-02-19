---
layout:     post
title:      C++ Singletons
date:       2020-02-16 22:19:00
author:     Blair Davidson
summary:    C++ Singleton Design Pattern
categories: C++
thumbnail:  heart
tags:
 - C++
 - Singleton
 - Design Patterns
---
The singleton design pattern is one of the original and oldest patterns from the mighty design patterns book, and also one of the
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
is also made private, but why? If the constructor was public we could then create instances of the object so it has to
be private. The destructor must be public so that later we can delete the object. 

So if the constructor is private, how do we create instances of the Singleton class. This is done through exposing a
static method instance which will create the Singleton. The instance itself is stored in a static class member
instance_. 

So what is the std::call_once and std::once_flag? Before going into to detail, what happens if to threads try to call
Singleton::instance? Well anything. We could end up with two objects being created and one of them will leak, or maybe
just one instance. std::call_once is from the header <mutex> and guarantees that the callable provided is only invoked
once. Hence in the Singleton example it is used to enforce only one call to new Singleton is called.

The is one not so obvious problem with the current program above. Lets revist that code.

{% highlight cpp %}
int main() {

    auto *inst = singleton::instance();

    return 0;
}

{% endhighlight %}

When this program runs its output is

{% highlight shell %}
singleton()

Process finished with exit code 0

{% endhighlight %}

Where is the call to ~Singleton()? Well we only have a pointer on the stack so no destructor is called. We need to use
our old friend RAII to help us out. Lets put a class together to help us out, scoped_singleton_deleter<T>. This class
takes a pointer of type T and ensures that when its destructor is called it guarantees to delete the object, which we
know is shorthard for ~T() followed by ::operator delete(*ptr). Therefore we cleanup afterwards.

Lets look at the code.

{% highlight cpp %}

template<typename T>
class scoped_singleton_deleter {
public:
    explicit scoped_singleton_deleter(T* ptr)
        : ptr_{ptr}
    {
    }
    scoped_singleton_deleter(scoped_singleton_deleter const&) = delete;
    scoped_singleton_deleter& operator=(scoped_singleton_deleter const&) = delete;
    scoped_singleton_deleter(scoped_singleton_deleter&&) = delete;
    scoped_singleton_deleter& operator=(scoped_singleton_deleter&&) = delete;
    ~scoped_singleton_deleter() {
        delete ptr_;
    }
private:
    T* ptr_{};
};

{% endhighlight %}

Now lets update main and see how this all hangs togther.

{% highlight cpp %}
int main() {

    auto *inst = singleton::instance();
    scoped_singleton_deleter<singleton> _{inst};

    return 0;
}

{% endhighlight %}

Lets now look at the output and see if this all works.

{% highlight shell %}
singleton()
~singleton()

Process finished with exit code 0
{% endhighlight %}

So here we have it. It all works. But what are some of the downsides of the Singleton design pattern. Well firstly once
the Singleton is created, we can access the object anywhere in the program with Singleton::instance(). This makes it
nice and easy for us but all makes it a nightmare to test code that uses the Singleton. We have a hidden dependency that
is rampant throughout the codebase. Also it is essentially a global variable and it makes code much harder to reason
about.

In future articles we will look at variations of this pattern.
