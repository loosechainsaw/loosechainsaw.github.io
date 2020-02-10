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

One of the biggest issues that arise in C++ is who owns a particular pointer. 

For example consider the simple function declaration below.

{% highlight cpp %}
int* function();
{% endhighlight %}

The caller of this function does not know at first glance whether then need to call delete on the pointer or whether they are just borrowing the pointer and library will take care of releasing the resource. If the caller deletes the resource when they are not supposed to bad things will happen most likely
and we will run into the land of undefined behaviour. If we were supposed to call delete then we have leaked the resource. The only thing we can really hope for is either the documentation is good or we can read the source to determine the right course of action.

If this leaves you feeling pretty unsatisfied read on. We can help improve on this by making this
explicit in our own code. Enter observable_ptr<T>. This is a user defined type that models the 
notion of a pointer that is borrowed and someone else is responsible for the cleanup. We are mearly
just observers.

Before we jump in for those who have not overloaded the * (dereference) and -> (arrow) operators this will be explained in depth as we go.

{% highlight cpp %}
#include <cassert>

template<typename T>
class observable_ptr {
public:
    friend bool operator==(observable_ptr<T> const& lhs, observable_ptr<T> const& rhs);
    friend bool operator!=(observable_ptr<T> const& lhs, observable_ptr<T> const& rhs);
    explicit observable_ptr(T* ptr = nullptr)
        : ptr_{ptr}
    {
    }
    observable_ptr(observable_ptr const&) = default;
    observable_ptr& operator=(observable_ptr const&) = default;
    T& operator*() noexcept;
    T const& operator*() const noexcept;
    T* operator->() noexcept;
    T const* operator->() const noexcept;
    bool empty() const noexcept;
    explicit operator bool() const noexcept;
    ~observable_ptr() = default;
private:
    T* ptr_{}; // default ptr to nullptr
};

template<typename T>
bool operator==(observable_ptr<T> const& lhs, observable_ptr<T> const& rhs) {
    return lhs.ptr_ == rhs.ptr_;
}

template<typename T>
bool operator!=(observable_ptr<T> const& lhs, observable_ptr<T> const& rhs) {
    return !(lhs == rhs);
}

template<typename T>
T& observable_ptr<T>::operator*() noexcept {
    assert(ptr_ != nullptr);
    return *ptr_;
}

template<typename T>
T const& observable_ptr<T>::operator*() const noexcept {
    assert(ptr_ != nullptr);
    return *ptr_;
}

template<typename T>
T* observable_ptr<T>::operator->() noexcept {
    assert(ptr_ != nullptr);
    return ptr_;
}

template<typename T>
T const* observable_ptr<T>::operator->() const noexcept {
    assert(ptr_ != nullptr);
    return ptr_;
}

template<typename T>
bool observable_ptr<T>::empty() const noexcept {
    return ptr_ == nullptr;
}

template<typename T>
observable_ptr<T>::operator bool() const noexcept {
    return ptr_ != nullptr;
}
{% endhighlight %}
