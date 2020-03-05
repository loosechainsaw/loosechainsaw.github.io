---
layout:     post
title:      C++ Polymorphic Storage
date:       2020-02-23 22:19:00
author:     Blair Davidson
summary:    C++ Polymorphic Storage
categories: C++
thumbnail:  heart
tags:
 - C++
 - Operator new
 - Placement new
 - Buffers
---
# Introduction
In this post, we will look at creating a polymorphic storage buffer. This is a class which uses an array bytes to store a user specified number of types,  which are derived from a user specified base type. The contents of the buffer in the storage class can change over the course of the program.

As the polymorphic buffer can store objects of various sizes, the storage provided must be the size of the of the largest type. Also the storage must be aligned to that of the type with the largest alignment requirement. There are two two traits, max_size and max_alignment, which are variadic templates which obviously finds the the max size and alignment of the list of types provided. The storage used is that of std::aligned_storage, which provides a nested typedef called type which is of the provided size and alignment. 

The library also provides support for std::unique_ptr. The benefit of this is that it is guaranteed to destroy the constructed type when the unique_ptr either goes out of scope or is reset. This also helps make the users code base exception safe and much simpler.

# Code

{% highlight cpp %}

#include <cstring>
#include <type_traits>
#include <memory>
#include <string>

namespace storage {

    namespace traits {

        template<typename... Ts>
        struct max_alignment;

        template<>
        struct max_alignment<> {
        public:
            static constexpr size_t value = 0;
        };

        template<typename T, typename... Ts>
        struct max_alignment<T, Ts...> {
        private:
            static constexpr size_t previous_max = max_alignment<Ts...>::value;
        public:
            static constexpr size_t value = previous_max > alignof(T) ? previous_max : alignof(T);
        };

        template<typename... Ts>
        struct max_size;

        template<>
        struct max_size<> {
        public:
            static constexpr size_t value = 0;
        };

        template<typename T, typename... Ts>
        struct max_size<T, Ts...> {
        private:
            static constexpr size_t previous_max = max_size<Ts...>::value;
        public:
            static constexpr size_t value = previous_max > sizeof(T) ? previous_max : sizeof(T);
        };
    }

    template<typename TBase, typename... Ts>
    class polymorphic_storage {
    public:
        polymorphic_storage() noexcept = default;
        polymorphic_storage(polymorphic_storage const &) = delete;
        polymorphic_storage &operator=(polymorphic_storage const &) = delete;
        template<typename TDerived, typename... Args>
        TDerived *construct(Args &&... args)
        noexcept(noexcept(TDerived{std::forward<Args>(args)...}))
        {
            if (!empty_)
                return nullptr;

            auto result = new(&storage_) TDerived{std::forward<Args>(args)...};
            empty_ = false;

            return result;
        }
        void destroy(void *ptr) noexcept {
            if (empty_)
                return;

            reinterpret_cast<TBase *>(ptr)->~TBase();
            empty_ = true;
        }
        ~polymorphic_storage() {
            destroy(&storage_);
        }

    private:
        typename std::aligned_storage<traits::max_size<Ts...>::value, traits::max_alignment<Ts...>::value>::type storage_{};
        bool empty_{true};
    };

    template<typename TStorage>
    class polymorphic_storage_deleter {
    public:
        explicit polymorphic_storage_deleter(TStorage* storage = nullptr) noexcept
                : storage_{storage}
        {
        }
        void operator()(void* ptr) noexcept {
            storage_->destroy(ptr);
        }
    private:
        TStorage* storage_{};
    };

    template<typename TBase, typename... Args>
    using polymorphic_unique_t = std::unique_ptr<TBase, polymorphic_storage_deleter<polymorphic_storage<TBase, Args...>>>;

    template<typename T,  typename TBase, typename... TDerived, typename... Args>
    polymorphic_unique_t<TBase, TDerived...> make_unique_polymorphic(polymorphic_storage<TBase, TDerived...>& storage, Args&&... args )
    noexcept(noexcept(T{std::forward<Args>(args)...}))
    {
        polymorphic_storage_deleter<polymorphic_storage<TBase, TDerived...>> _{&storage};
        auto constructed = storage.template construct<T>(std::forward<Args>(args)...);

        polymorphic_unique_t<TBase, TDerived...> result{constructed, _};

        return result;
    }

}

namespace examples {

    class shape {
    public:
        shape() = default;
        shape(shape const &) = delete;
        shape& operator=(shape const &) = delete;
        virtual void print() = 0;
        virtual ~shape() = default;
    };

    class circle : public shape {
    public:
        void print() override {
            printf("%s\n", name_.c_str());
        }
    private:
        std::string name_{"circle"};
    };

    class square : public shape {
    public:
        void print() override {
            puts("square\n");
        }
    };

    class rectangle : public shape {
    public:
        void print() override {
            puts("rectangle\n");
        }
    };
}

int main() {

    using namespace storage;
    using namespace examples;

    using shape_polymorphic_storage_t = polymorphic_storage<shape, circle, square, rectangle>;
    shape_polymorphic_storage_t storage{};

    auto s = make_unique_polymorphic<circle>(storage);
    s->print();

    return 0;
}

{% endhighlight %}

# Discussion
The example program above use a heirarchy of shape classes which can be stored in the polymorphic buffer. The first type parameter of the buffer is the base type, which is shape in the example, and the second type parameter is a variadic type, which contains a list of derived types of shape. These are circle, square and rectangle. The buffer storage value can change over the course of the program and we manage the lifetime of the derived type with a std::unique_ptr which has a custom deleter. If you are in an embedded environment a type like this can be useful as you often are not permitted to use the heap. You just need to be careful around swapping the contents of the buffer and having an old std::unique_ptr around which thinks it is pointing to something else. Beware.

Until next time.
