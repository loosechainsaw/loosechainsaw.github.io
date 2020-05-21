---
layout:     post
title:      C++ class template specialization
date:       2020-03-01 22:19:00
author:     Blair Davidson
summary:    C++ class template specialization
categories: C++
thumbnail:  heart
tags:
 - C++
 - Templates
 - STL
 - Specialization
---
# Introduction
Class templates in C++ can specialized for particular combination of template arguments. There a quite a few reasons why the author may choose to specialize a class template including, fixing incorrect behaviour for a particular type argument, optimizing for a type argument, and others.

In this article we will show both types of specialization, explicit and partial and look at an example of using this with a data structure and explore how specialization allow use to provide traits, allowing other developers to get meta data like inforation of types.

The data structure we will use is a fixed size stack and show how to specialize for pointer types and void. We will also develop traits for is_const<T>, is_pointer<T>, remove_reference<T> and is_void<T>.

Lets look at the fixed_stack<T, N> data structure first.


s


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
- Provide operator !=
OutputIterator concept:
-Can write to the current element of a iterator
-When combined with a ForwardIterator etc, the iterator is considered mutable
ForwardIterator concept:
-Basically InputIterator but allows multiple passes
BidirectionIterator
-Refinement of ForwardIterator
-ForwardIterator than can move in both directions
-Supports --i
RandomAccessIterator
-Refinement of BidirectionalIterator
-Can access anywhere in the underlying source in constant time
-Supports i += n, i + n, i - n, i -= n
-Supports i < k, i <= k, i > k, i >= k
{% endhighlight %}

As an example of a more sophisticated data structure, lets look at a linked list that can be traversed in a single direction, forwards. The forward_list<T> class, defines two iterator classes, forward_list_iterator and const_forward_list_iterator. Lets take a look at the code.

{% highlight cpp %}
#include <iostream>

namespace datastructures {

    template<typename T>
    class forward_list {
    public:

        using value_type = T;
        using reference = T&;
        using const_reference = T const&;
        using pointer = T*;
        using const_pointer = T const*;
        using size_type = size_t;
        using difference_type = ptrdiff_t ;

        struct node {
            value_type value{};
            node* next{};
        };

        class forward_list_iterator {
        public:
            using value_type = T;
            using reference = T&;
            using const_reference = T const&;
            using pointer = T*;
            using const_pointer = T const*;
            using size_type = size_t;
            using difference_type = ptrdiff_t ;
            using iterator_category = std::forward_iterator_tag;

            forward_list_iterator() noexcept = default;
            explicit forward_list_iterator(node* ptr)
            : ptr_{ptr}
            {
            }
            forward_list_iterator(forward_list_iterator const&) noexcept = default;
            forward_list_iterator& operator=(forward_list_iterator const&) noexcept = default;
            forward_list_iterator& operator++() noexcept {
                ptr_ = ptr_->next;
                return *this;
            }
            forward_list_iterator operator++(int) noexcept {
                auto temp = *this;
                this->operator++();
                return temp;
            }
            reference operator*() noexcept {
                return ptr_->value;
            }
            const_pointer operator->() noexcept {
                return &(ptr_->value);
            }
            bool operator==(forward_list_iterator const& rhs) const noexcept {
                return ptr_ == rhs.ptr_;
            }
            bool operator!=(forward_list_iterator const& rhs) const noexcept {
                return ptr_ != rhs.ptr_;
            }
            ~forward_list_iterator() = default;
        private:
            node* ptr_{};
        };

        class const_forward_list_iterator {
        public:
            using value_type = T;
            using reference = T&;
            using const_reference = T const&;
            using pointer = T*;
            using const_pointer = T const*;
            using size_type = size_t;
            using difference_type = ptrdiff_t ;
            using iterator_category = std::forward_iterator_tag;

            const_forward_list_iterator() noexcept = default;
            explicit const_forward_list_iterator(node* ptr)
                : ptr_{ptr}
            {
            }
            const_forward_list_iterator(const_forward_list_iterator const&) noexcept = default;
            const_forward_list_iterator& operator=(const_forward_list_iterator const&) noexcept = default;
            const_forward_list_iterator& operator++() noexcept {
                ptr_ = ptr_->next;
                return *this;
            }
            const_forward_list_iterator operator++(int) noexcept {
                auto temp = *this;
                this->operator++();
                return temp;
            }
            const_reference operator*() noexcept {
                return ptr_->value;
            }
            const_pointer operator->() noexcept {
                return &(ptr_->value);
            }
            bool operator==(const_forward_list_iterator const& rhs) const noexcept {
                return ptr_ == rhs.ptr_;
            }
            bool operator!=(const_forward_list_iterator const& rhs) const noexcept {
                return ptr_ != rhs.ptr_;
            }
            ~const_forward_list_iterator() = default;
        private:
            node* ptr_{};
        };

        using iterator = forward_list_iterator;
        using const_iterator = const_forward_list_iterator;


        forward_list() noexcept = default;
        forward_list(const forward_list&) = default;
        forward_list& operator=(const forward_list&) = default;
        forward_list(forward_list&&) noexcept;
        forward_list& operator=(forward_list&&) noexcept;
        void push_front(T const& v);
        void push_front(T&& v);
        void push_back(T const& v);
        void push_back(T&& v);
        void pop_front();
        void clear();
        iterator begin() noexcept {
            return iterator{head_};
        }
        iterator end() noexcept {
            return iterator{};
        }
        iterator cbegin() const noexcept {
            return const_iterator{head_};
        }
        iterator cend() const noexcept {
            return const_iterator{};
        }
        T& front() noexcept;
        T const& front() const noexcept;
        T& back() noexcept;
        T const& back() const noexcept;
        size_t size() const noexcept;
        bool empty() const noexcept;
        ~forward_list();
    private:
        node *head_{};
        node *tail_{};
        size_t size_{};
    };

    template<typename T>
    forward_list<T>::forward_list(forward_list&& rhs) noexcept
            : head_{rhs.head_},
              tail_{rhs.tail_},
              size_{rhs.size_}
    {
        rhs.head_ = nullptr;
        rhs.tail_ = nullptr;
        rhs.size_ = 0;
    }

    template<typename T>
    forward_list<T>& forward_list<T>::operator=(forward_list&& rhs) noexcept
    {
        clear();
        head_ = rhs.head_;
        tail_ = rhs.tail_;
        size_ = rhs.size_;

        rhs.head_ = nullptr;
        rhs.tail_ = nullptr;
        rhs.size_ = 0;
    }

    template<typename T>
    void forward_list<T>::push_front(const T &v)
    {
        head_ = new node{v, head_};
        if(!tail_)
            tail_ = head_;
        ++size_;
    }

    template<typename T>
    void forward_list<T>::push_front(T &&v) {
        head_ = new node{std::move(v), head_};
        if(!tail_)
            tail_ = head_;
        ++size_;
    }

    template<typename T>
    void forward_list<T>::push_back(const T &v) {
        auto n =  new node{v, nullptr};
        if(!tail_) {
            tail_ = n;
        }else{
            tail_->next = n;
            tail_= tail_->next;
        }
        ++size_;
    }

    template<typename T>
    void forward_list<T>::push_back(T&& v) {
        auto n =  new node{std::move(v), nullptr};
        if(!tail_) {
            tail_ = n;
        }else{
            tail_->next = n;
            tail_= tail_->next;
        }
        ++size_;
    }

    template<typename T>
    void forward_list<T>::pop_front() {
        if(empty()) return;

        auto next = head_->next;
        delete head_;
        head_ = next;

        if(head_ == nullptr)
            tail_ = head_;
        --size_;
    }

    template<typename T>
    void forward_list<T>::clear() {

        auto p = head_;

        while(p != nullptr) {
            auto t = p->next;
            delete p;
            p = t;
        }
        size_ = 0;
    }

    template<typename T>
    bool forward_list<T>::empty() const noexcept {
        return head_ == nullptr;
    }

    template<typename T>
    forward_list<T>::~forward_list() {
        clear();
    }

    template<typename T>
    size_t forward_list<T>::size() const noexcept {
        return size_;
    }

    template<typename T>
    T &forward_list<T>::front() noexcept {
        return head_->value;
    }

    template<typename T>
    T const &forward_list<T>::front() const noexcept {
        return head_->value;
    }

    template<typename T>
    T &forward_list<T>::back() noexcept {
        return tail_->value;
    }

    template<typename T>
    T const &forward_list<T>::back() const noexcept {
        return tail_->value;
    }
}

int main() {

    datastructures::forward_list<int> v;

    for(auto i = 0; i < 10; ++i){
        v.push_front(i);
    }

    datastructures::forward_list<int> z {std::move(v)};

    for(size_t i = 0, size = z.size(); i < size; ++i){
        std::cout << z.front() << "\n";
        z.pop_front();
    }

    std::cout << "================================\n";

    for(auto i = 0; i < 10; ++i){
        v.push_front(i);
    }
    
    std::for_each(std::begin(v), std::end(v), [](int z) {
        std::cout << z << "\n";
    });

    std::cout << std::endl;

    return 0;
}

{% endhighlight %}

The first thing we can observe it we define the iterators dereference operator, operator*() in terms of returning a reference to the value field of node structure. Secondly the arrow operator, operator->() is defined in terms of returning a pointer to the value field of node structure. This is important as when an algorithm, such as std::find need a value from the iterator we return the correct value for use in the algorithm. Also the prefix operator, operator++() is defined in terms of updating the the ptr to the current node to node->next. This is the logically way for the iterator of a list class to move through its collection of nodes. Finally we define equality, operator==() and inequality, operator!=() in terms of comparing node pointers. This again makes sense as we stop when we hit the end iterators address, which is nullptr, which matches how we would do this if we just wrote an algorithm to walk a list.

The iterators also define a typedef iterator_category. This is a requirement for iterators. This lets algorithms know what the iterator supports in terms of requirements. Algorithms can be customised to use more efficent algorithms if the iterator supports them. For example std::random_access_iterator_tag supports more operators then std::forward_iterator_tag. 

So this wraps up our introduction into iterators. I hope this helps. In the following article on iterators we will discuss iterator_traits. But before then there will be additional articles on the prerequisites such as template specialization and traits.

Until next time.
