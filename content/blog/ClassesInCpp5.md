+++
title = "Ring Buffer Series Part 5 — const_iterator"
date = "2026-05-11T12:00:42+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "classes"]

[images]
    featured_image = "/img/blog/ringbuffer5.jpg"
+++

![class](/img/blog/ringbuffer5.jpg)
[Previously](ClassesInCpp4) we expanded our RingBuffer by implementing its own iterator class along with iterator traits. This allowed us to make it work with range-based for loops and most algorithms in the C++ Standard Library, but it still has a limitation.
```cpp
void print_contents(const RingBuffer<int, 8>& buf)
{
    for(auto x : buf)
    {
        std::cout << x << ' ';
    }
}
```
The compiler will not allow it. Our `begin()` and `end()` are not `const` qualified, which means they can't be called on a `const` object. In this episode we will fix this by introducing `const_iterator`, an iterator that promises not to modify the elements it visits.

## What is a const_iterator?
A `const_iterator` is a type of iterator that provides read-only access to their elements. It is the STL equivalent of `pointer to const`. We can use it to move through the buffer (`++it`), but we can't use it to modify the value it refers to. Using them is a best practice as it ensures that functions that need to inspect data don't actually change by accident. Regularly they can be obtained by calling `cbegin()` and `cend()` or calling `begin()` and `end()` on a container that has been declared as `const`.

## const basics
The `const` keyword specifies that an object or value will remain unchanged. When we apply `const` to a variable, the variable needs to be initialized at declaration time and cannot be modified afterward. Consider:
```cpp
const int x{5}; //Declaring and initializing an integer with the value 5
x = 7;          //This is a compiler error because x is declared as const
```

## Why Does const Break Our Iterator?
Our `RingBuffer` broke because `begin()` and `end()` are not `const`-qualified member functions. A `const`-qualified member function promises not to modify the object it is called on, which is what allows it to be called on a `const` object. When the compiler sees `const RingBuffer<int, 8>& buf`, it only allows calls to member functions that make this promise. Our `begin()` and `end()` don't, so the compiler rejects them.

Before we look at the code, we need to see what actually changes. Recall that the `operator*` in our `iterator` is already marked `const`, but that `const` protects the iterator itself from being modified during dereferencing. It says nothing about the element. The return type is what controls that. Our `iterator` returns `T&`, allowing the caller to modify the element. Our `const_iterator` will return `const T&` instead, making the element read-only. Beyond that, only two other things change: the stored pointer becomes `const RingBuffer*` and the traits aliases use `const T*` and `const T&`. Everything else, the logical offset, modular arithmetic, `operator++`, `operator!=`, `operator==`, stays identical.

## Updating Our RingBuffer:
To define a `const_iterator` we need to create another nested class inside our `RingBuffer`, the new class is almost exactly the same as the `iterator`, but there will be some key differences.

```cpp
#ifndef RINGBUFFER_H
#define RINGBUFFER_H
#include <iostream>

template <typename T, std::size_t N>
class RingBuffer
{
public:
    class iterator {
    public:
        using iterator_category = std::forward_iterator_tag; //Required for min/max_element
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = T*;
        using reference = T&;

        T& operator*() const {
            return rb_->buffer_[(rb_->head_ + index_) % N];
        }

        iterator& operator++() {
            ++index_; //Move to the next logical element
            return *this;
        }

        bool operator!=(const iterator& other) const {
            return index_ != other.index_; //Compare logical positions
        }

        //We also add the equality operator
        bool operator==(const iterator& other) const {
            return rb_ == other.rb_ && index_ == other.index_;
        }
    private:
        friend class RingBuffer; //Allow RingBuffer to construct iterators
        iterator(RingBuffer* rb, std::size_t index) : rb_{ rb }, index_{ index } {}

        RingBuffer* rb_;    //Pointer to the parent container
        std::size_t index_; //Logical offset from the head
    };

    class const_iterator {
    public:
        using iterator_category = std::forward_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = const T*;   //Points to a constant element
        using reference = const T&;  //Returns a constant reference

        //Returns const reference
        const T& operator*() const {
            return rb_->buffer_[(rb_->head_ + index_) % N];
        }

        //Notice that all other operators remain the same
        const_iterator& operator++() {
            ++index_; 
            return *this;
        }

        bool operator!=(const const_iterator& other) const {
            return index_ != other.index_; 
        }

        bool operator==(const const_iterator& other) const {
            return rb_ == other.rb_ && index_ == other.index_;
        }
    private:
        friend class RingBuffer;
        const_iterator(const RingBuffer* rb, std::size_t index) : rb_{ rb }, index_{ index } {}

        const RingBuffer* rb_;
        std::size_t index_;
    };

    //Existing non-const iterators for modification
    iterator begin() { return iterator(this, 0); }    //First element
    iterator end() { return iterator(this, count_); } //Past-the-end sentinel

    //Const overload for read-only contexts like `print_contents`
    const_iterator begin() const { return const_iterator(this, 0); }
    const_iterator end() const { return const_iterator(this, count_); }

    //Explicitly constant iterators
    const_iterator cbegin() const { return begin(); }
    const_iterator cend() const { return end(); }

    RingBuffer();       //Constructor
    ~RingBuffer();      //Destructor

    void print_buffer() const;              //Outputs the buffers contents to the screen
    void push_back(const T& new_element);   //Adds a new element into the buffer
    void pop_front();                       //Deletes the oldest element from the buffer
    bool empty() const;                     //Is the buffer empty
    bool full() const;                      //Is the buffer full
    const T& front() const;                 //Retrieves the oldest element
    std::size_t size() const;               //Retrieves the number of elements in the buffer

private:
    T buffer_[N];                              //Notice that now it take N and not 8.
    std::size_t head_;                         //Index of oldest element (read position)
    std::size_t tail_;                         //Index of next write position
    std::size_t count_;                        //Total number of elements in the buffer
};

template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer() : buffer_{}, head_{}, tail_{}, count_{} {}

template <typename T, std::size_t N>
RingBuffer<T, N>::~RingBuffer() {}

template <typename T, std::size_t N>
void RingBuffer<T, N>::print_buffer() const
{
    std::size_t index{ head_ };
    for (std::size_t i = 0; i < count_; i++) {
        std::cout << buffer_[index] << ' ';
        index++;
        if (index >= N) {
            index = 0;
        }
    }
    std::cout << '\n';
}

template <typename T, std::size_t N>
void RingBuffer<T, N>::push_back(const T& new_element)
{
    if (count_ == N) {
        //Buffer full - We are about to overwrite oldest element
        head_++;
        if (head_ >= N) {
            head_ = 0;
        }
    }
    else {
        count_++;
    }
    buffer_[tail_] = new_element;
    tail_++;
    if (tail_ >= N) {
        tail_ = 0;
    }
}

template <typename T, std::size_t N>
void RingBuffer<T, N>::pop_front()
{
    if (count_ == 0) {
        // Buffer empty, nothing to remove
        return;
    }
    head_++;
    if (head_ >= N) {
        head_ = 0;
    }
    count_--;
}

template <typename T, std::size_t N>
bool RingBuffer<T, N>::empty() const
{
    return count_ == 0;
}

template <typename T, std::size_t N>
bool RingBuffer<T, N>::full() const
{
    return count_ == N;
}

template <typename T, std::size_t N>
const T& RingBuffer<T, N>::front() const
{
    assert(!empty());
    return buffer_[head_];
}

template <typename T, std::size_t N>
std::size_t RingBuffer<T, N>::size() const
{
    return count_;
}
#endif // RINGBUFFER_H
```
As noted, the duplication between `iterator` and `const_iterator` is intentional. Production codebases sometimes use a single template parameterized on constness to generate both types, but the trade-off is added complexity. For our purposes, two small classes with clear intent are easier to understand and maintain.

We have seen the compiler pick `const_iterator` automatically when the buffer is `const`, but what if you have a mutable buffer and want read-only access? This is what `cbegin()` and `cend()` aim to solve. They are member functions used to obtain constant iterators from a container. They are important to enforce const correctness, so that functions do not change data by accident. `cbegin()` returns a `const_iterator` pointing to the first element of the container. `cend()` returns a `const_iterator` pointing to one position after the last valid element. 

## How Does It Work?
The compiler uses a process called overload resolution to decide which version of `begin()` to call by looking at a hidden `this` pointer of the object. The `this` pointer is a hidden parameter passed to every non-static member function, it allows the member function to access data member and other methods of the specific object it is currently operating on. It is normally used to return a self-reference `return *this`. In our `RingBuffer` we used `this` inside `begin()` and `end()` to pass the current buffer's address to the `iterator` constructor making sure that it knows what container it belongs to.

If the object is `const`, the `this` pointer is a pointer to const (`const T*`). C++ forbids `const` objects from calling non-const member functions, so at this point the regular `begin()` is disqualified and the compiler has to pick the `begin() const` overload. If the object is mutable that means that its `this` pointer is a regular `T*`. While a mutable object is allowed to call a `const` function, the non-const version is considered a better match during resolution, so the compiler will select the regular `begin()` to allow for potential modifications.

## Seeing it in Action
Now, we can try the troublesome code from the beginning of the post:
```cpp
void print_contents(const RingBuffer<int, 8>& buf)
{
    for(auto x : buf)
    {
        std::cout << x << ' ';
    }
}

void show_stats(const RingBuffer<int, 8>& buf)
{
    auto max_pos{ std::max_element(buf.begin(), buf.end()) };
    auto min_pos{ std::min_element(buf.begin(), buf.end()) };
    std::cout << "Max: " << *max_pos << '\n';
    std::cout << "Min: " << *min_pos << '\n';
}

int main()
{
    RingBuffer<int, 8> buffer;
    for (int i = 0; i < 10; ++i)
    {
        buffer.push_back(i);
    }

    // const reference — uses const_iterator automatically
    std::cout << "Calling print_contents:\n";
    print_contents(buffer);
    std::cout << '\n';
    // STL algorithms through a const reference
    show_stats(buffer);

    // cbegin/cend — read-only access on a mutable buffer
    auto it{ std::find(buffer.cbegin(), buffer.cend(), 7) };
    if (it != buffer.cend())
    {
        std::cout << "Found via cbegin/cend: " << *it << '\n';
    }

    return 0;
}
```
And the output is!
```
Calling print_contents:
2 3 4 5 6 7 8 9 
Max: 9
Min: 2
Found via cbegin/cend: 7
```

## What We Accomplished
Our RingBuffer has come a long way. In [part 1](ClassesInCpp) we started with the basics of classes and functions, then in [part 2](ClassesInCpp2) we built a concrete implementation, then in [part 3](ClassesInCpp3) we generalized it with templates and gave it iterator support in [part 4](ClassesInCpp4) and it now handles `const` correctness. The buffer works with range-based for loops, STL algorithms and `const` references, the same procedures that standard library containers follow.

## What's Next?
Our buffer is correct, but it is not yet complete. Right now, every element we store is copied into the buffer, which is fine for an `int`, but not great for types such as `std::string`. In the next post we will tackle move semantics and round out the interface with a few utility methods to make our RingBuffer ready for real-world use.