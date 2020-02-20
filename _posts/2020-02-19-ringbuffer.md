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

{% highlight cpp %}

#include <iostream>
#include <type_traits>
#include <algorithm>

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
            reference operator*() noexcept {
                return (*source_)[index_];
            }
            template<bool Z = C, typename std::enable_if<(Z), int>::type* = nullptr>
            const_reference operator*() const noexcept {
                return (*source_)[index_];
            }
            template<bool Z = C, typename std::enable_if<(!Z), int>::type* = nullptr>
            reference operator->() noexcept {
                return &((*source_)[index_]);
            }
            template<bool Z = C, typename std::enable_if<(Z), int>::type* = nullptr>
            const_reference operator->() const noexcept {
                return &((*source_)[index_]);
            }
            self_type& operator++() noexcept {
                index_ = ++index_ % N;
                ++count_;
                return *this;
            }
            self_type operator++(int) noexcept {
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

        ring_buffer() noexcept(std::is_nothrow_default_constructible_v<value_type>) = default;
        ring_buffer(ring_buffer const&)
            noexcept(std::is_nothrow_default_constructible_v<value_type> &&
                     std::is_nothrow_copy_constructible_v<value_type>) = default;
        ring_buffer& operator=(ring_buffer const&)
            noexcept(std::is_nothrow_copy_constructible_v<value_type>) = default;
        template<typename U>
        void push_back(U&& value) {
            push_back(std::forward<U>(value), std::bool_constant<Overwrite>{});
        }
        void pop_front(){
            if(empty())
                return;

            if(!empty())
                --size_;

            tail_ = ++tail_ %N;
        }
        [[nodiscard]] reference back() noexcept { return elements_[std::clamp(head_, 0UL, N - 1)]; }
        [[nodiscard]] const_reference back() const noexcept {
            return const_cast<self_type*>(back)->back();
        }
        [[nodiscard]] reference front() noexcept { return elements_[tail_]; }
        [[nodiscard]] const_reference front() const noexcept {
            return const_cast<self_type*>(this)->front();
        }
        reference operator[](size_type index) noexcept {
            return elements_[index];
        }
        const_reference operator[](size_type index) const noexcept {
            return const_cast<self_type *>(this)->operator[](index);
        }
        [[nodiscard]] iterator begin() noexcept { return iterator{this, tail_, 0};}
        [[nodiscard]] iterator end() noexcept { return iterator{this, head_, size_};}
        [[nodiscard]] const_iterator cbegin() const noexcept { return const_iterator{this, tail_, 0};}
        [[nodiscard]] const_iterator cend() const noexcept { return const_iterator{this, head_, size_};}
        [[nodiscard]] bool empty() const noexcept { return size_ == 0; }
        [[nodiscard]] bool full() const noexcept { return size_ == N; }
        [[nodiscard]] size_type capacity() const noexcept { return N; }
        ~ring_buffer() = default;
    private:
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
            elements_[head_] = std::forward<U>(value);
            head_ = ++head_ %N;

            if(full())
                tail_ = ++tail_ % N;

            if(!full())
                ++size_;
        }
        T elements_[N]{};
        size_type head_{};
        size_type tail_{};
        size_type size_{};
    };

}

int main() {

    using namespace buffers;
    ring_buffer<int, 3> b1;

    for(auto i = 0; i < 9; ++i){
        b1.push_back(i + 1);
    }

    decltype(b1) b2 = b1;

    b2.pop_front();
    b2.pop_front();

    std::for_each(b2.cbegin(), b2.cend(), [](int value) {
        std::cout << value << std::endl;
    });

    return 0;
}


{% endhighlight %}

