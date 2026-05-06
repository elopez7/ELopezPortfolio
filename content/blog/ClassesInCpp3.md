+++
title = "Ring Buffer Series Part 3 — Templates: One Buffer, Any Type, Any Size"
date = "2026-05-04T12:00:42+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "classes"]

[images]
    featured_image = "/img/blog/class.jpeg"
+++

![class](/img/blog/class.jpeg)
In [Part 2](ClassesInCpp2) we built a buffer that works, however it is stuck with `unsigned int` and size 8, but what if we wanted to use it with `std::string` or custom types like game events, or only store the last 4 frames? Do we write a buffer per class? We could, but that would be a nightmare, wouldn't it?

# Meet Templates
In C++, templates are a powerful mechanism for generating code. A template is a blueprint that the compiler uses to generate actual code. Think about it this way, a class is a blueprint for a thing (a type of thing), templates are blueprints for classes.

## Function Templates
Function templates are blueprints the compiler uses to create concrete functions that allow you to perform operations across multiple data types without repeating code.
The `template` keyword is used to introduce a function template, following the keyword there is a list of parameters that the function can take.
Consider the following:
```cpp
template <typename T>
T max_of(T a, T b){
    return (a > b) ? a : b;
}
```
In the code above the `T` is a type template parameter, a placeholder that will be replaced by the appropriate type when we call this function. In the following code, we take the function for a test ride.
```cpp
int main()
{
    int intA{5};
    int intB{10};
    float floatA{20.5};
    float floatB{20.9};

    int largestInt{max_of(intA, intB)};
    float largestFloat{max_of(floatA, floatB)};

    std::cout << "The largest int is: " << largestInt << '\n';     //largestInt is 10
    std::cout << "The largest float is: " << largestFloat << '\n'; //largestFloat is 20.9
}
```

## Class Templates
Just like a class is a blueprint for creating things, a class template is a blueprint for creating classes. It is commonly used for container types like `std::vector` or `std::array` and they allow the generated classes to store any kind of data type. We will use this knowledge to our advantage by giving our ring buffer the ability to store something more than just `unsigned int`. We start by replacing the specific type with a template type parameter, normally known as `T`.
```cpp
#ifndef RINGBUFFER_H
#define RINGBUFFER_H

template <typename T>
class RingBuffer
{
public:
    RingBuffer();       //Constructor
    ~RingBuffer();      //Destructor
    
    void print_buffer() const;       //Outputs the buffers contents to the screen
    void push_back(T new_element);   //Adds a new element into the buffer
    void pop_front();                //Deletes the oldest element from the buffer
    bool empty() const;              //Is the buffer empty
    bool full() const;               //Is the buffer full
    T front() const;                 //Retrieves the oldest element
    std::size_t size() const;          //Retrieves the number of elements in the buffer

private:
    T buffer_[8];                    //The actual buffer container, in this case an array.
    std::size_t head_;                         //Index of oldest element (read position)
    std::size_t tail_;                         //Index of next write position
    std::size_t count_;                //Total number of elements in the buffer
};
#endif // RINGBUFFER_H
```
For the most part this is what we would need to do to make the RingBuffer into a template, but there is a small detail. If you remember in the original buffer we had divided code between a header file `.h` and an implementation file `.cpp` which usually it is a good thing to do, but not for templates.
If I were to put the definitions in the `.cpp` file and then tried to compile, then I would likely get a linker error (`unresolved external symbols`).

From the compiler's point of view, templates are not "real" code because nothing has been instantiated with a specific type. In other words, the compiler cannot see the definitions of the template because it needs to know what specific type to associate it with. What are we to do, then?
Well, a common solution is to put the definitions in the header file, usually inside the class itself. Consider the following code:
```cpp
#ifndef RINGBUFFER_H
#define RINGBUFFER_H
#include <iostream>

template <typename T>
class RingBuffer
{
public:
    RingBuffer(): buffer_{}, head_{}, tail_{}, count_{} {}

    ~RingBuffer(){}

    void print_buffer() const
    {
        std::size_t index{head_};
        for(std::size_t i = 0; i < count_; i++){
            std::cout << buffer_[index] << ' ';
            index++;
            if(index >= 8){
                index = 0;
            }
        }
        std::cout << '\n';
    }

    void push_back(T new_element)
    {
        if(count_ == 8){
            //Buffer full - We are about to overwrite oldest element
            head_++;
            if(head_ >= 8){
                head_ = 0;
            }
        }else{
            count_++;
        }

        buffer_[tail_] = new_element;
        tail_++;
        if(tail_ >= 8){
            tail_ = 0;
        }
    }

    void pop_front()
    {
        if(count_ == 0){
            // Buffer empty, nothing to remove
            return;
        }

        head_++;
        if(head_ >= 8){
            head_ = 0;
        }
        count_--;
    }

    bool empty() const
    {
        return count_ == 0;
    }

    bool full() const
    {
        return count_ == 8;
    }

    T front() const
    {
        assert(!empty());
        return buffer_[head_];
    }

    std::size_t size() const
    {
        return count_;
    }

private:
    T buffer_[8];                   //The actual buffer container, in this case an array.
    std::size_t head_;              //Index of oldest element (read position)
    std::size_t tail_;              //Index of next write position
    std::size_t count_;             //Total number of elements in the buffer
};
#endif // RINGBUFFER_H
```
The above normally is acceptable when you have a simple type or simple functions, but when you work in a large project with a large codebase, this approach becomes a nightmare to read and to follow, luckily we have an alternative, we can put the definitions in the same `.h` file, but outside the class, consider:
```cpp
#ifndef RINGBUFFER_H
#define RINGBUFFER_H
#include <iostream>

template <typename T>
class RingBuffer
{
public:
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
    T buffer_[8];                           //The actual buffer container, in this case an array.
    std::size_t head_;                      //Index of oldest element (read position)
    std::size_t tail_;                      //Index of next write position
    std::size_t count_;                     //Total number of elements in the buffer
};

template <typename T>
RingBuffer<T>::RingBuffer(): buffer_{}, head_{}, tail_{}, count_{} {}

template <typename T>
RingBuffer<T>::~RingBuffer(){}

template <typename T>
void RingBuffer<T>::print_buffer() const
{
    std::size_t index{head_};
    for(std::size_t i = 0; i < count_; i++){
        std::cout << buffer_[index] << ' ';
        index++;
        if(index >= 8){
            index = 0;
        }
    }
    std::cout << '\n';
}

template <typename T>
void RingBuffer<T>::push_back(const T& new_element)
{
    if(count_ == 8){
        //Buffer full - We are about to overwrite oldest element
        head_++;
        if(head_ >= 8){
            head_ = 0;
        }
    }else{
        count_++;
    }
    buffer_[tail_] = new_element;
    tail_++;
    if(tail_ >= 8){
        tail_ = 0;
    }
}

template <typename T>
void RingBuffer<T>::pop_front()
{
    if(count_ == 0){
        // Buffer empty, nothing to remove
        return;
    }
    head_++;
    if(head_ >= 8){
        head_ = 0;
    }
    count_--;
}

template <typename T>
bool RingBuffer<T>::empty() const
{
    return count_ == 0;
}

template <typename T>
bool RingBuffer<T>::full() const
{
    return count_ == 8;
}

template <typename T>
const T& RingBuffer<T>::front() const
{
    assert(!empty());
    return buffer_[head_];
}

template <typename T>
std::size_t RingBuffer<T>::size() const
{
    return count_;
}
#endif // RINGBUFFER_H

```
While this approach is a little bit more verbose, it is the closest we can get to having the declarations separated from the definitions. In more recent C++ standards there are other ways to achieve this, but we won't get into them because that's out of the scope of this series. You will also notice that `T front()` has become `const T& front()`. Originally, we were copying `unsigned int` which for the compiler is a trivial operation, however now that we are using templates, `T` can be anything, even a `std::string` which is wasteful to copy, so now instead of copying we are using a reference. This is also useful for when `T` becomes an incredibly expensive type to copy. This same concept also applies to `void push_back(T new_element)` which has now become `void push_back(const T& new_element)`.

Great, our RingBuffer is able to take any type now and it should work, but it still has a problem.
It will only ever support 8 elements or whatever number we decided, whoever wants to use it will not be able to change that number at all, but fear not dear reader for C++ offers us a way to make it possible to work with different sizes. We do this by using a Non-Type Template Parameter. This allows us to pass a value `std::size_t` as a template argument. 

What is `size_t`? I hear you ask. `std::size_t` is an unsigned integer type guaranteed to be large enough to represent the size of any object on the current platform. On a 64-bit system, `unsigned int` might only be 32 bits — too small to index very large arrays — while `std::size_t` will be 64 bits.

We can now update our class with the following:
```cpp
#ifndef RINGBUFFER_H
#define RINGBUFFER_H
#include <iostream>

template <typename T, std::size_t N>
class RingBuffer
{
public:
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
# Seeing it in Action
With that in place anyone who wants to use this buffer can do so in the following manner.
```cpp
int main()
{
    RingBuffer<int, 10> my_buffer;              //A buffer of 10 integers
    RingBuffer<float, 6> big_buffer;            //A buffer of 6 floats
    RingBuffer<std::string, 5> string_buffer;   //A buffer of 5 strings

    for (auto i = 0; i < 10; ++i)
    {
        my_buffer.push_back(i);
    }
    for (auto i = 0; i < 100; ++i)
    {
        big_buffer.push_back(i+0.1f);
    }
    for (auto i = 0; i < 5; ++i)
    {
        string_buffer.push_back("Position: " + std::to_string(i));
    }
    std::cout << "Printing int buffer\n";
    my_buffer.print_buffer();
    std::cout << "Printing float buffer\n";
    big_buffer.print_buffer();
    std::cout << "Printing string buffer\n";
    string_buffer.print_buffer();
}
```
The output would be:
```
Printing int buffer
0 1 2 3 4 5 6 7 8 9 
Printing float buffer
94.1 95.1 96.1 97.1 98.1 99.1 
Printing string buffer
Position: 0 Position: 1 Position: 2 Position: 3 Position: 4 
```

# Next Steps
Our buffer stores anything and it can be of any size, but it doesn't fully behave like any of the built in types from the `STL`, we can't use something like `for (auto x : buffer)` and we will take care of that in the next post.

# References / Sources
- [Class Templates - cppreference](https://en.cppreference.com/cpp/language/class_template)
- [Non-type template parameters - cppreference](https://en.cppreference.com/cpp/language/template_parameters)
- [std::size_t - cppreference](https://en.cppreference.com/cpp/types/size_t)