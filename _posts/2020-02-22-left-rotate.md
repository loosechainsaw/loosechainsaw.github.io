---
layout:     post
title:      C++ Ring buffer
date:       2020-02-22 22:19:00
author:     Blair Davidson
summary:    C++ Ring buffer
categories: C++
thumbnail:  heart
tags:
 - C++
 - Iterators
 - STL
 - rotate
---
# Introduction
In this post we will look at rotating a datastucture like an array to the left. This means moving each element one place to the left, with the first item being moved into the last position. We basically start at the first item to the last item and copy the next element over the top. This is means at any position the value will be that of the next position. However because we copy over the top with the next element we would lose the first element if we do not store it in a temporary variable. Also we need to make sure that we stop if the next element is equal to one past the end of the array. The algorithm is a generic algorithm and is parameterized on the concept of forward iterators. This means that pretty much any stl compliant datastructure can use this algorithm.

# Walk through
As an example given a array of size 3, lets walk through some examples so see how it works.


{% highlight plaintext %}
-------
|1|2|3| 
-------
 
 Set temp = 1; and move everything down

-------
|2|3|3| 
-------

 Now copy temp to the last position

-------
|2|3|1| 
-------

{% endhighlight %}

# Code


{% highlight cpp %}

template<typename TForwardIterator>
TForwardIterator left_rotate(TForwardIterator begin, TForwardIterator end) {
    if(begin == end)
        return begin;

    auto last = begin;
    auto temp = *last;

    for(auto start = begin; start != end; ++start){
        auto next = std::next(start);
        last = start;

        if(next == end)
            break;

        *start = *next;
    }

    if(last != begin)
        *last = temp;

    return last;
}

{% endhighlight %}

So from the code, lets discuss. We first check if begin == end, is it empty, if so there is nothing to do so return begin. Then we store a temp of the element at begin, and set last, which refers to the actual last element in the range to begin. We then walk through the range, and check if the next elment is the end, and if so exit, otherwise we dereference the iterator and set its value to the next elements value. Also we update the last to be the current iterator. After we are done we check if the range is not empty and set the value of last to the temp value. Voila.

Lets look at it in action.


{% highlight cpp %}

int main() {

    std::vector<int> v{1,2,3,4,5};
    left_rotate(std::begin(v), std::end(v));
    left_rotate(std::begin(v), std::end(v));
    left_rotate(std::begin(v), std::end(v));

    std::for_each(std::begin(v), std::end(v), [](int element){
        std::cout << element << " ";
    });

    std::cout << std::endl;

    return 0;
}
{% endhighlight %}
