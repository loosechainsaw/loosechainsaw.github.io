---
layout:     post
title:      Fun overloading * and -> in C++
date:       2020-02-08 22:19:00
author:     Blair Davidson
summary:    Playing with the * and -> operators in C++
categories: C++
thumbnail:  heart
tags:
 - C++
 - pointers
 - operator overloading
---

The C++ language allows the developer to overload the unary operator * dereference operator and -> member access
operator.

The unary dereference operator return a reference to some type T and must be a member function of the class and can be
overloaded on const and non const. So why bother? Well there are a couple of examples from the standard library which we
will use to set the scene. But before jumping into examples, think about what the * operator does with pointers. It
dereferences the pointer and returns a lvalue expression referencing to the object that the pointers value stores. This
is important as one of the goals of creating your own overloaded operators should be do something that is obvious to the
user, the principal of least astonishment.

Lets look at std::optional<T> from the standard library introduced in C++ 17 to start with. The purpose of
std::optional<T> is to model the notion of something that may or may not hold a value of type T. This is different to
that of a pointer as std::optional<T> holds the value of T within its state and not just a pointer to an object on the
free store, this is mandated by the standard. Lets have a quick look at a quick example of using std::optional<T> with a
contrived example of division. When we attempt to divide by zero we return an empty optional.

{% highlight cpp %}
#include <iostream>
#include <optional>

std::optional<double> divide(int numerator, int denominator) noexcept {
    std::optional<double> result; 
    if(denominator != 0) {
        result = numerator / denominator; 
    }
    return result;
}

int main(int argc, char** argv) {

    auto result1 = divide(6, 2);

    if(result1.has_value())
        std::cout << "value of 6 / 2 is " << *result1 << std::endl;


    auto result2 = divide(6, 0);

    if(!result2.has_value())
        std::cout << "Cannot divide 6 / 0" << std::endl;

    return 0;
}


{% endhighlight %}

Looking at the above code, after testing result1 for having a value, result1 is dereferenced to obtain a reference to
the stored value. This is a great example of overloading the * operator. The example below is an example optional<T>
type. This is a basic implementation to show only how the * and -> operators work.


