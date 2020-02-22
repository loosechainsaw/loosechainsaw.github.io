---
layout:     post
title:      C++ Ring buffer
date:       2020-02-19 22:19:00
author:     Blair Davidson
summary:    C++ Ring buffer
categories: C++
thumbnail:  heart
tags:
 - C++
 - Ring buffer
 - Data structures
---
# Introduction
A Ring buffer is a datastructure that is a fixed size queue which wraps around when it reaches the end. It does not need to expand when full, it simply overwrites the the oldest element. The ring buffer can be implemented as an array, with two pointers or indexes, one for the head and one for the tail. The head is where the next value will be placed and the tail is a pointer or index to the oldest element in the ring buffer. When we want to see the oldest element or front of the queue we just look at the tail. If we want to pop the front the queue we just increment the tail and decrement the size. When we want to add a new element to the ringbuffer we add it to the position of head and then increment head and the size. So what happens when we are at capacity? Well we have a design choice, overwrite the oldest element or drop the new value on the floor. The ring buffer I have developed has a template parameter to allow the user to select the desire behaviour. If you are at capacity and and the head is in the same position as the tail you overwrite the oldest element and then increment the tail, which is now the second oldest item. As the head and tail are incremented, the author of the ring buffer has to make sure the indexes or pointers do not fall off the end of the array. This is achieved by modulus with the size of the array. 

# Walk through
As an example given a Ringbuffer of size 3, lets walk through some examples so see how it works.


{% highlight plaintext %}
-------
| | | | An empty ring buffer 
-------
^
H/T
-------
|1| | | First element pushed into the ring buffer 
-------
 ^ ^
 T H
-------
|1|2| | Second element pushed into the ring buffer 
-------
 ^   ^
 T   H
-------
|1|2|3| Third element pushed into the ring buffer 
-------
 ^
 T/ H

{% endhighlight %}

The ring buffer is now full. As mentioned earlier we can either, drop new elements on the floor. This is the easy case. We just check if size == capacity and return if this is true. The other case is to overwrite existing elements. Lets look at adding another element. When the fourth element is push the last element is overwriten and then the tail is bumped as is the head. As remember the head point to the next position to write to.

{% highlight plaintext %}
-------
|4|2|3| Fourth element pushed into the ring buffer 
-------
   ^
   H/T
 
{% endhighlight %}

# Code

The ring buffer below provides the following features:
1. Generic
2. User can decide whether to overwrite or drop elements when full
3. Reduces the requirements on T, T does not have to be default constructible
4. Has iterator support

{% highlight cpp %}

#include <iostream>
#include <type_traits>
#include <algorithm>
#include <cstring>
#include <vector>

namespace buffers {

    template<typename T, size_t N, bool Overwrite = true>
    class ring_buffer;

    namespace detail {
        template<typename T, size_t N, bool C, bool Overwrite>
        class ring_buffer_iterator {
            using buffer_t = typename std::conditional<!C, ring_buffer<T, N, Overwrite>*, ring_buffer<T, N, Overwrite> const*>::type;
        public:
            using self_type = ring_buffer_iterator<T, N, C, Overwrite>;
            using value_type = T;
            using reference = T&;
            using const_reference = T const&;
            using pointer = T*;
            using const_pointer = T const*;
            using size_type = size_t;
            using difference_type = ptrdiff_t;
            using iterator_category = std::forward_iterator_tag;

            ring_buffer_iterator() noexcept = default;
            ring_buffer_iterator(buffer_t source, size_type index, size_type count) noexcept
                    : source_{source},
                      index_{index},
                      count_{count}
            {
            }
            ring_buffer_iterator(ring_buffer_iterator const& ) noexcept = default;
            ring_buffer_iterator& operator=(ring_buffer_iterator const& ) noexcept = default;
            template<bool Z = C, typename std::enable_if<(!Z), int>::type* = nullptr>
            [[nodiscard]] reference operator*() noexcept {
                return (*source_)[index_];
            }
            template<bool Z = C, typename std::enable_if<(Z), int>::type* = nullptr>
            [[nodiscard]] const_reference operator*() const noexcept {
                return (*source_)[index_];
            }
            template<bool Z = C, typename std::enable_if<(!Z), int>::type* = nullptr>
            [[nodiscard]] reference operator->() noexcept {
                return &((*source_)[index_]);
            }
            template<bool Z = C, typename std::enable_if<(Z), int>::type* = nullptr>
            [[nodiscard]] const_reference operator->() const noexcept {
                return &((*source_)[index_]);
            }
            [[nodiscard]] self_type& operator++() noexcept {
                index_ = ++index_ % N;
                ++count_;
                return *this;
            }
            [[nodiscard]] self_type operator++(int) noexcept {
                auto result = *this;
                this->operator*();
                return result;
            }
            [[nodiscard]] size_type index() const noexcept {
                return index_;
            }
            [[nodiscard]] size_type count() const noexcept {
                return count_;
            }
            ~ring_buffer_iterator() = default;
        private:
            buffer_t source_{};
            size_type index_{};
            size_type count_{};
        };

        template<typename T, size_t N, bool C, bool Overwrite>
        bool operator==(ring_buffer_iterator<T,N,C,Overwrite> const& l,
                        ring_buffer_iterator<T,N,C,Overwrite> const& r) noexcept {
            return l.count() == r.count();
        }

        template<typename T, size_t N, bool C, bool Overwrite>
        bool operator!=(ring_buffer_iterator<T,N,C,Overwrite> const& l,
                        ring_buffer_iterator<T,N,C,Overwrite> const& r) noexcept {
            return l.count() != r.count();
        }
    }

    template<typename T, size_t N, bool Overwrite>
    class ring_buffer {
        using self_type = ring_buffer<T, N, Overwrite>;
    public:
        static_assert(N > 0, "ring buffer must have a size greater than zero.");

        using value_type = T;
        using reference = T&;
        using const_reference = T const&;
        using pointer = T*;
        using const_pointer = T const*;
        using size_type = size_t;
        using iterator = detail::ring_buffer_iterator<T, N, false, Overwrite>;
        using const_iterator = detail::ring_buffer_iterator<T, N, true, Overwrite>;

        ring_buffer() noexcept = default;
        ring_buffer(ring_buffer const& rhs) noexcept(std::is_nothrow_copy_constructible_v<value_type>)
        {
            copy_impl(rhs, std::bool_constant<std::is_trivially_copyable_v<T>>{});
        }
        ring_buffer& operator=(ring_buffer const& rhs) noexcept(std::is_nothrow_copy_constructible_v<value_type>) {
            if(this == &rhs)
                return *this;

            destroy_all(std::bool_constant<std::is_trivially_copyable_v<T>>{});
            copy_impl(rhs, std::bool_constant<std::is_trivially_copyable_v<T>>{});

            return *this;
        }
        template<typename U>
        void push_back(U&& value) {
            push_back(std::forward<U>(value), std::bool_constant<Overwrite>{});
        }
        void pop_front() noexcept{
            if(empty())
                return;

            destroy(tail_, std::bool_constant<std::is_trivially_destructible_v<value_type>>{});

            --size_;
            tail_ = ++tail_ %N;
        }
        [[nodiscard]] reference back() noexcept { return reinterpret_cast<reference>(elements_[std::clamp(head_, 0UL, N - 1)]); }
        [[nodiscard]] const_reference back() const noexcept {
            return const_cast<self_type*>(back)->back();
        }
        [[nodiscard]] reference front() noexcept { return reinterpret_cast<reference >(elements_[tail_]); }
        [[nodiscard]] const_reference front() const noexcept {
            return const_cast<self_type*>(this)->front();
        }
        [[nodiscard]] reference operator[](size_type index) noexcept {
            return reinterpret_cast<reference >(elements_[index]);
        }
        [[nodiscard]] const_reference operator[](size_type index) const noexcept {
            return const_cast<self_type *>(this)->operator[](index);
        }
        [[nodiscard]] iterator begin() noexcept { return iterator{this, tail_, 0};}
        [[nodiscard]] iterator end() noexcept { return iterator{this, head_, size_};}
        [[nodiscard]] const_iterator cbegin() const noexcept { return const_iterator{this, tail_, 0};}
        [[nodiscard]] const_iterator cend() const noexcept { return const_iterator{this, head_, size_};}
        [[nodiscard]] bool empty() const noexcept { return size_ == 0; }
        [[nodiscard]] bool full() const noexcept { return size_ == N; }
        [[nodiscard]] size_type capacity() const noexcept { return N; }
        void clear() noexcept{
            destroy_all(std::bool_constant<std::is_trivially_destructible_v<value_type>>{});
        }
        ~ring_buffer() {
            clear();
        };
    private:
        void destroy_all(std::true_type) { }
        void destroy_all(std::false_type) {
            while(!empty()) {
                destroy(tail_, std::bool_constant<std::is_trivially_destructible_v<value_type>>{});
                tail_ = ++tail_ % N;
                --size_;
            }
        }
        void copy_impl(self_type const& rhs, std::true_type) {
            std::memcpy(elements_, rhs.elements_, rhs.size_ * sizeof(T));
            size_ = rhs.size_;
            tail_ = rhs.tail_;
            head_ = rhs.head_;
        }
        void copy_impl(self_type const& rhs, std::false_type) {
            tail_ = rhs.tail_;
            head_ = rhs.head_;
            size_ = rhs.size_;

            try {
                for (auto i = 0; i < size_; ++i)
                    new( elements_ + ((tail_ + i) % N)) T(rhs[tail_ + ((tail_ + i) % N)]);
            }catch(...) {
                while(!empty()) {
                    destroy(tail_, std::bool_constant<std::is_trivially_destructible_v<value_type>>{});
                    tail_ = ++tail_ % N;
                    --size_;
                }
                throw;
            }
        }
        template<typename U>
        void push_back(U&& value, std::true_type) {
            push_back_impl(std::forward<U>(value));
        }
        template<typename U>
        void push_back(U&& value, std::false_type) {
            if(full() && !Overwrite)
                return;
            push_back_impl(std::forward<U>(value));
        }
        template<typename U>
        void push_back_impl(U&& value) {

            if(full())
                destroy(head_, std::bool_constant<std::is_trivially_destructible_v<value_type>>{});

            new(elements_ + head_ ) T{std::forward<U>(value)};
            head_ = ++head_ %N;

            if(full())
                tail_ = ++tail_ % N;

            if(!full())
                ++size_;
        }
        void destroy(size_type index, std::true_type) noexcept { }
        void destroy(size_type index, std::false_type) noexcept {
            reinterpret_cast<pointer >(&elements_[index])->~T();
        }
        typename std::aligned_storage<sizeof(T), alignof(T)>::type elements_[N]{};
        size_type head_{};
        size_type tail_{};
        size_type size_{};
    };

}
{% endhighlight %}

Below shows how the ring buffer is used.

{% highlight cpp %}
int main() {
    using namespace buffers;
    ring_buffer<std::vector<int>, 3> b1;

    for(auto i = 0; i < 9; ++i){
        b1.push_back(std::vector<int>{i, i + 1, i + 2});
    }

    auto b2 = b1;

    ring_buffer<std::vector<int>, 3> b3;
    b3 = b2;
    b3.pop_front();

    std::for_each(b3.cbegin(), b3.cend(), [](std::vector<int> value) {
        std::cout << value[0] << " " << value[1] << " " << value[2] << std::endl;
    });

    return 0;
}

{% endhighlight %}
