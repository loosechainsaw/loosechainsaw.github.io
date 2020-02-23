---
layout:     post
title:      C++ Left Rotation
date:       2020-02-22 22:19:00
author:     Blair Davidson
summary:    C++ Left Rotation
categories: C++
thumbnail:  heart
tags:
 - C++
 - Iterators
 - STL
 - rotate
---
# Introduction
In this post, we will look at an algorithm which rotates a data structure's elements to the left. The reader should also be able to apply what they have learnt after reading this article to develop an algorithm, which rotates a datastructure's elements to the right.

Rotating a data structure, like an array or vector, involves shifting each element one place to the left. The first element however will be shifted into the last position of the data structure. The last element in the data structure has already been shifted one place to the left. 

The process starts at the first element and proceeds to the last element, copying the subsequent element into the current elements place, thus overwriting it. However because we overwrite the current element in each iteration, with that of the subsequent element, we need to store a temporary copy of the first element, so we can use it to replace the last element in the data structure. Special consideration also needs to be given when the last element of the data structure is reached, since we can not access the subsequent element.

The rotate_left algorithm algorithm presented in this article, uses iterators, like the rest of the STL. Future articles will cover how iterators work and how to implement data structures which use iterators, which are also compatible with the STL. The ring buffer article on this site, is an example of an STL compatible data stucture, which will be used in a future article to exmplain in depth the STL requirements for data structures and iterators. Suffice for now, iterators can be considered a generalization of pointers, which allow any compatible data structure to use the STL algorithms, or algorithms designed with iterator support in mind.

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

In the left_rotate algorithm shown above, the first point to note is that we do not need to do any work if the begin == end. The left_rotate algorithm presented returns the position of the last element in the range. This useful for the caller of the algorithm as they can check if the result is equal to the end iterator. If so no rotation was performed.

Secondly the algorithm stores a copy of the element at the beginning. This as mentioned previously is to ensure we do not lose a copy of this element. 

We also track which iterator value is equal to that of the the last addressable element in the range. This is one before the end, as iterators model a half open interval, [begin, end), and end is one past the end of the elements. This is needed as we need to overwrite this element with a copy of temp, which is the value of the element at the beginning.

The for loop, iterates through the range and until we reach the end iterator. In the body of the loop we save a copy of the iterator which represents the next position in the range. the variable last is also updated with the value of the current iterator and we exit if the next element is equal to the end iterator. This will be the actual last position in the range. Finally we update the element at the current position, start, with the value of the next iterator.

When the loop completes will just need to copy the value in temp to the location which the iterator last is refering to. This only need to be done however if we actually had to step through the array, which only happens if the size of the range is greater than one. And we return last, indicated to the caller that the range is firstly not empty, and that the range has now been shifted one to the left.

If we wanted to shift a range, say n to the left, this function could be wrapped in another function in a while loop and called until it has been performed n times.

Just for comparision, I will present some clojure code, which is a lisp. The clojure code has been contributed by a friend of mine, [Crispin Wellington](https://github.com/retrogradeorbit), which does the equivalent, but is in my opinion really elegant. Definately shows how beautiful functional programming languages are.

{% highlight clojure %}
(defn leftrotate [[h & t]]
  (concat t [h]))
{% endhighlight %}

Until next time.
