---
layout:     post
title:      C++ Iterators
date:       2020-03-01 22:19:00
author:     Blair Davidson
summary:    C++ Iterators
categories: C++
thumbnail:  heart
tags:
 - C++
 - Iterators
 - STL
 - algorithms
---
# Introduction
Iterators are pervasive in C++. The Standard Template Library (STL) algorithms are based on the iterator concept, along with many third party libraries such as boost and poco, to name a few. The quickest route to success with iterators based on my own experience with them is a look at a simple example and then the mechanics of how they actually work. 

Take the following algorithm. Given an array of unsorted elements find an element matching a specific value. Its is important to note the array is unsorted. So we cannot use binary_search, we can however just walk the array and check each element and return the index of element if it is a match. 

One of the choices we may wish to make is that of making the algorithm a template function. This seems to be a good choice as we can then process elements of any type that is equalable, providing support for == and !-. Lets see how this looks.

{% highlight cpp %}
template<typename T, int N>
int find( T (&input)[N], T const& value) {
    int result = -1;
    for(auto i = 0; i < N; ++i)
        if(input[i] == value) {
            result = i;
            break;
        }
    return result;
}
{% endhighlight %}

So what are the issues with the above approach? It works for all arrays right? Well it does work for all arrays of T, but what if we want to find a value in a std::vector<T>? Lets look at that?


{% highlight cpp %}
template<typename T>
int find( std::vector<T> const& input, T const& value) {
    int result = -1;
    for(auto i = 0; i < input.size(); ++i)
        if(input[i] == value) {
            result = i;
            break;
        }
    return result;
}

{% endhighlight %}

Well that works right? its so close to the array example though, just a few minor changes. What if we wanted to find a value in a std::string? Well I think you are starting to get the point here. How can we generalize this algorithm to work for any data structure? Enter iterators.

Iterators are pointer like constructs that allow you traverse a data structure, much like traversing an array. Algorithms are written in terms of iterators and data structures provide the algorithms iterators into their data structure to allow the traversal. Lets look a really basic example. Lets implement this with our own cut down version of std::array<T, N>. 

{% highlight cpp %}
#include <iostream>
#include <algorithm>

template<typename T, size_t N>
class array {
public:
    using value_type = T;
    using reference = T&;
    using const_reference = T const&;
    using pointer = T*;
    using const_pointer = T const*;
    using difference_type = ptrdiff_t;
    using size_type = size_t;
    using iterator = T*;
    using const_iterator = T*;

    array() = default;
    array(array const &) = default;
    array& operator=(array const &) = default;
    reference front() noexcept { return *elements_; }
    const_reference front() const noexcept { return *elements_; }
    reference back() noexcept { return elements_[N - 1]; }
    const_reference back() const noexcept { return elements_[N - 1]; }
    size_type size() const noexcept { return N; }
    reference operator[](size_type index) noexcept { return elements_[index];}
    const_reference operator[](size_type index) const noexcept { return elements_[index];}
    iterator begin() noexcept { return elements_; }
    const_iterator cbegin() const noexcept { return elements_; }
    iterator end() noexcept { return elements_ + N; }
    const_iterator cend() const noexcept { return elements_ + N; }
    ~array() = default;
    value_type elements_[N]{};
};

int main() {

    array<int, 3> v = {1,2,3};

    std::for_each(v.begin(), v.end(), [](int e) {
        std::cout << e << std::endl;
    });

    return 0;
}

{% endhighlight %}

Running the program gives the output:

{% highlight plaintext %}
1
2
3

Process finished with exit code 0
{% endhighlight %}

Lets look at the array<T, N> class. Firstly it has a bunch of type alias at the top. These type aliases are expected of containers that are STL compliant. The 2 we are interested in though are that of iterator and const_iterator. Firstly for the array<T, N> these are nothing but alias to pointer and const pointer types, which makes sense for a type that models an array<T, N>. For more complicated classes which we will look at later in this article, the iterator type aliases wont be merly pointer aliases, but aliases to user defined types. 

The next thing to notice is the begin / cbegin and end / cend functions. These are also required as these are used to provide the start and end of the iterator range. Remember an iterator provides access to the internals of a datastructure and allows us to traverse it. An important point to make is that the STL is modelled on half open intervals. If you do not know what this we will go over the concept now to fill you in.

# Intervals

Ok what are intervals and what is this notation. An iterval is a range of numbers. The notation allows us to specify whether the end numbers are included in the range. Lets look at some examples.

{% highlight plaintext %}

The numbers 1 - 4. Inclusive of both ends [1,4]. The [ or ] means include.
The numbers 1 - 4, not including the 5. [1,4). The ) or ( means not included.
The numbers 1 - 4, not including the 1. (1, 4].
The numbers 1 - 4, not including 1 or 4. (1, 4).

{% endhighlight %}

So how does this relate to the STL. Well the range of elements in a data structure that a pair of iterators, one for the start and one for the end is [begin, end). So this means that the end is one past the end. If you look at what end() returns in the array<T, N> returns it is one past the end. There are a bunch of benefits for using one past the end which we will duscuss later on. You just need to remember this for now.

# Back to the code

In the previous example we passed the results of begin() and end() to the function std::for_each(). This algorithm works on a pair of iterators. For each element pointed to by the iterator, the call back is called, printing the element to the console.

Lets get a feel for this by implementing std::for_each ourselves.

{% highlight cpp %}
template<class InputIt, class F>
F for_each(InputIt first, InputIt last, F f)
{
    for (; first != last; ++first) {
        f(*first);
    }
    return f; 
}

{% endhighlight %}

std::for_each takes a pair of iterators, first and last, of type InputIt and a Callback function f of type F. You can see that this function requires that the iterators support !=, ++ and the deference operator *.  This is not a problem for out simple array<T, N> which just uses a pointer type as iterators, but how about a linked list? It can not be modelling a simply a pointer type. Lets have a look at the requirements of iterator types.

# Concepts

A concept is a set of requirements on a type. Both syntactic, what expressions should be allowed, and semantic requirements, what should happen as a result of the expressions. A type that satisfies the requirement of a concept is said to model the concept. A refinement is a concept that adds additonal requirements to an existing concept.

Some of the basic concepts from the STL are:

{% highlight plaintext %}
DefaultConstructible: a type can be default constructed. 
MoveConstructible: a type is move constructable.
CopyConstructible: a type is copy constructable.
MoveAssignable: a type is move assignable.
CopyAssignable: a type is copy assignable.
EqualityComparable. a type can be compared for equality and inequality.
{% endhighlight %}

# Iterator Concepts

Iterators in the STL also form a heirarchy of concepts. Iterators the support being able to read from and moved forwards only once are input iterators. The reason for being restricted to be read from only once is the iterator might be something like a network stream, which by nature can be consumed only once. Lets examine Input Iterators in more details to see the operations which must be supported. 

{% highlight plaintext %}
Iterator concept:
- Refinement of CopyConstructible
- Refinement of CopyAssignable
- Refinement of Destructible
- Refinement of Swappable
- Has typedefs for value_type, difference_type, reference, pointer, and iterator_category

InputIterator concept:
- Refinement of Iterator
- Suports !=
- Suports operator++() and operator++(int). 
- Supports operator*() and operator->()

{% endhighlight %}

Lets take our new understanding of iterators and implement that find method we spoke about at the start of the article. We can implement find() with a pair of Input Iterators and an additional template parameter to represent the value we wish to find in the range modelled by the input iterators provided. 


{% highlight cpp %}
template<class InputIt, class T>
InputIt find(InputIt first, InputIt last, const T& value)
{
    for (; first != last; ++first) {
        if (*first == value) {
            return first;
        }
    }
    return last;
}
{% endhighlight %}

Lets examine find(). It loops through the range, incrementing the iterator at each pass and checks if the value matches the supplied value and returns the position of the iterator if a match occurs. Otherwise it returns the end, which indicates we did not find anything.

So far you maybe scratching your head as to why bother with iterators. I mean i we have done is provide pointers. Well lets create our own linked list class with only links to the next node. We can only move forwards.

{% highlight cpp %}

{% endhighlight %}

I dont want to explain how a linked list works, but lets examine the iterator class it provides. The first thing to noice is the implementation of operator++(). It updates the element that the iterator pointers to by moving to the next node with element_ = element_->next; It also works like how the traditional prefix unary ++ operator works. It updates the value immediately and returns a reference to that value. The operator(int) models the postfix operator, thus it makes a copy of itself first and then updates itself and returns a the old version of itself.

The iterator also implements the operator*() and operator->() functions. The operator*() works by returning a reference to the actual value contained in the node pointed to by element_, whilst operator->() returns a pointer to the object contained in the node structure.

To see how the range of elements presented by the linked list are navigatible. Look at what begin() and end() return. begin() return the pointer contained in head_, whilst end() return nullptr. This makes sense when you think about how an algorithm like find() works. It has to stop when we get to end(), which is what the first != last check serves. It also needs to move through the data structure providing the iterators, which is what ++first does. Remember it just moves forward with element_ = element_->next; Finally it has to check if the value matches the supplied value. We have this with our operator*() implementation which returns the value contained in the node structure by reference.

Until next time.
