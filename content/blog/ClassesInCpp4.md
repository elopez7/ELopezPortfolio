+++
title = "Ring Buffer Series Part 5 — Iterators: Walking the Ring"
date = "2026-05-05T12:00:42+02:00"
draft = true
type = "blog"
tags = ["programming", "posts", "articles", "C++", "classes"]

[images]
    featured_image = "/img/blog/class.jpeg"
+++

![class](/img/blog/class.jpeg)
In [Part 3](ClassesInCpp3) we expanded on the original buffer and it now supports any type and any size. Our RingBuffer almost behaves like any of the `STL` types, such as `std::vector` and I say almost. Consider the following code:
```cpp
int main()
{
    RingBuffer<int, 10> my_buffer;

    for (auto x : my_buffer)      //This line produces an error.            
    {
        my_buffer.push_back(x);
    }

    std::cout << "Printing int buffer\n";
    my_buffer.print_buffer();
}
```
This code should work, but if we actually try to run it we will get a compiler error.
```
error: 'begin' was not declared in this scope
```
So, what's wrong? Recall in a previous post we used code similar to the following.
```cpp
int main()
{
    std::vector<double> numbers{1,2,3,4,5,6,7,8,9,10};
   // We use std::max_element
    auto max_pos{std::max_element(numbers.begin(), numbers.end())};
    double max_value{*max_pos};

    // We use std::min_element
    auto min_pos{std::min_element(numbers.begin(), numbers.end())};
    double min_value{*min_pos};
}
```
Take a look at how `max_pos` and `min_pos` are calculated. You will notice that use of two functions. `.begin()` and `.end()`. Now, the compiler error starts to make sense, our template class does not have a `begin()` function of its own and even if it had one, there is no `.end()` function either. This means that there is still work to do!

## The Range-Based for Loop
The range-based for loop was introduced in C++ 11. Its goal is to simplify iterating over collections.
When we write something like `for(auto i : collection)`, the compiler rewrites it into a traditional `for` loop that uses iterators to navigate the range. The produced code would looks something like this.
```cpp
{
    auto __begin = collection.begin();
    auto __end = collection.end();
    for(; __begin != __end ++__begin){
        i = *__begin;
        //Statement
    }
}
```
There are 5 requirements for us to be able to implement this functionality in our RingBuffer class.
1.  **`.begin()`**: The compiler first looks for a member function (or a global function via ADL) that returns an iterator pointing to the **first element**.
2.  **`.end()`**: It similarly requires a function returning a **past-the-end sentinel**, which marks the position immediately after the last valid element.
3.  **`operator!=`**: The loop uses this to compare the current position (`__begin`) against the sentinel (`__end`) to determine if the iteration should continue.
4.  **`operator++`**: The **prefix increment** operator is required to advance the iterator to the next logical element in the sequence.
5.  **`operator*`**: The **dereference** operator must be implemented so the loop can access and assign the value at the current position to your loop variable.

I know this is a lot, but don't worry, we will expand on these. Let's begin by defining one word I have been using quite lot lately; `iterator` what is that?

## What is an Iterator?
An iterator is an object that acts as a generic abstraction for accessing and traversing a container. Iterators are what allow one algorithm to work in different kinds of containers likes `std::vector` or `std::list` without having to know the way they are implemented, for example an algorithm can sum values in a sequence whether those values are stored in a `std::vector` or `linked_list`. 
A sequence is typically defined by a pair of iterators: `begin()` which points to the first element and `end()` which points to "past the end" which is one position after the last element.
Now that we know what an iterator is, let's start implementing one for our RingBuffer.

## Nested Iterator Implementation
A way to implement the iterator is by declaring it as a nested class inside the RingBuffer. An advantage to this approach is that is that it allows to maintain a clean and standard interface. Our ring buffer iterator needs to behave in such a way that the iterator can navigate from `head_` to `tail_` while handling the array's wrap around. Consider the following:
```cpp
#ifndef RINGBUFFER_H
#define RINGBUFFER_H
#include <iostream>

template <typename T, std::size_t N>
class RingBuffer
{
public:
    class iterator{
    public:
        T& operator*() const{
            return rb_->buffer_[(rb_->head_ + index_) % N];
        }

        iterator& operator++(){
            ++index_; //Move to the next logical element
            return *this;
        }

        bool operator!=(const iterator& alarm) const{
            return index_ != alarm.index_; //Compare logical positions
        }
    private:
        friend class RingBuffer; //Allow RingBuffer to construct iterators
        iterator(RingBuffer* rb, std::size_t index) : rb_{rb}, index_{index}{}

        RingBuffer* rb_;    //Pointer to the parent container
        std::size_t index_; //Logical offset from the head
    };

    //Container requirements to yield iterators
    iterator begin() {return iterator(this, 0);}    //First element
    iterator end() {return iterator(this, count_);} //Past-the-end sentinel

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
    T buffer_[N];                           //Notice that now it take N and not 8.
    std::size_t head_;                      //Index of oldest element (read position)
    std::size_t tail_;                      //Index of next write position
    std::size_t count_;                     //Total number of elements in the buffer
};

//Now, everywhere where we had the hardcoded 8 can be updated to use N

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
## Operator Overload
There is a lot to unpack here, let's go over it a little bit slower.
Inside the RingBuffer class we are creating an iterator class. Then we have three functions inside the iterator class, notice the `operator` keyword (`operator*`, `operator++` and `operator!=`).
The `operator` keyword is used to declare an overloas operator function. This allows us to redefine how standard C++ operators (`+`,`==` or `[]`) behave when used with custom types. Here is what it allows you to do. In short Operator Overloading tells the compiler that a function provides a custom implementation for a symbol. You start with `operator` and then add the symbol you want to change `operator+`, for example and now we can tell the compiler that `+` will be used to subtract instead of add. There are other things that `operator` allows us to do, but that it out of the scope of what we want to achieve.

# Dereference: T& operator*()
Consider the following:
```cpp
T& operator*() const{
    return rb_->buffer_[(rb_->head_ + index_) % N];
}
```
The `*` allows the iterator access to the element at the iterator's current position. The RingBuffer is a special case, so we had to define a special operation. We must provide logical to physical mapping, this means that the "first" element is not always at array index `0`