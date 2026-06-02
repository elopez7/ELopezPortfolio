+++
title = "Ring Buffer Series Part 7 — Move Semantics - Don't copy, steal"
date = "2026-05-16T12:00:42+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "classes"]

[images]
    featured_image = "/img/blog/ringbuffer7.jpg"
+++

![class](/img/blog/ringbuffer7.jpg)
In [part 6](ClassesInCpp6) we closed by pointing towards move semantics. In this post we will tackle the inefficiency of unnecessary copying, which happens when we create a temporary object, copy it into a container and then destroy the original. We acknowledged that this process is wasted work and in our RingBuffer it happens because our `push_back` implementation only supports copying, even then the source is a temporary object.
Consider the following:
```cpp
struct LoudHeavy{
    int id;
    int* data;
    static constexpr std::size_t data_size{1000};

    LoudHeavy(int i) : id{i}, data{new int[data_size]}{
        std::cout << " [Ctro] Created " << id << '\n';
    }

    //Instrumented Copy Constructor
    LoudHeavy(const LoudHeavy& other) : id(other.id), data(new int[data_size]){
        std::memcpy(data, other.data, data_size * sizeof(int));
        std::cout << " [COPY] Duplicating " << id << " (" << data_size * sizeof(int) << " bytes)\n" ;
    }

    ~LoudHeavy(){
        delete[] data;
        std::cout << " [Dtor] Destroyed " << id << "\n";
    }
};

int main()
{
    RingBuffer<LoudHeavy, 2> rb;
    std::cout << "Pushing Temporary:\n";
    rb.push_back(LoudHeavy{1}); //Passing an rvalue (temporary)
    std::cout << "Done\n";   
    return 0;
}
```
And the output:
```
Pushing Temporary:
 [Ctro] Created 1
 [COPY] Duplicating 1 (4000 bytes)
 [Dtor] Destroyed 1
Done
 [Dtor] Destroyed 1
```
Here is what is happening, we are first constructing a temporary object on the stack, then we are copying that temporary object into the RingBuffer's memory and then the temporary object is destroyed because the copying is done. We are essentially paying for two separate allocations for the `int` array, one for the temporary object and one for the buffer element. We spent CPU cycles manually moving 4000 bytes from the stack to the buffer, only to delete the source bytes immediately after. On some systems, these unnecesary allocations can be hundreds of times more expensive than simple function calls. This is an overhead that can be avoided if we implement move semantics.

# What Does "move" Mean?
For our context, we define "move" as transferring ownership of resources, such as heap memory or file handles from a source object to a target object instead of duplicating them. The generic explanation is as follows; the target "steals" the internal pointers and state from the source, the source is then reset to "valid, but unspecified" state, usually empty or null, so that its destructor doesn't release the stolen memory. For our `RingBuffer`, "moving" is what makes the difference between allocating 4000 bytes for a fresh `int` array copy versus just copying an 8-byte pointer. This is the basis of move semantics.
![Copy vs move](/img/blog/CopyvsMove.jpg)
# Move Semantics
Move semantics is one of the greatest features in the C++ language. Introduced by the C++11 standard, it is designed to avoid unnecessary, expensive copying of temporary objects or objects that are no longer needed. To understand move semantics in a deeper level, there are some concepts that we must visit first:

## Value Categories
In C++, every expression is characterized by two properties, a type and a value category. These categories will allow us to know if an object's resources can be "stolen".

### lvalue (Left Values)
An lvalue represents an object that occupies a specific, identifiable location in memory, we can typically take their address using the `&` operator and if they are not `const` we can assign values to them. Consider the following:
```cpp
int main()
{
    //1. Named variables are lvalues
    int x{42};
    int* ptr{&x}; // can take the address of lvalue 'x'

    //2. Dereferenced pointers are lvalues
    *ptr = 100;   // '*ptr' refers to the actual object 'x'

    //3. Array elements are lvalues
    int arr[10]{1,2,3};
    for(int& i : arr){
        i = 10;   // Each array element has a specific memory location
    }

    //4. lvalue references are lvalues
    int& ref{x};
    ref = 50;     // `ref` is an alias for the lvalue `x`

    //5. String literals are lvalues (unlike any other literal)
    const char* str{"Hi"};  //: "Hi" is a string literal and has an address
}
```

### rvalue (Right Values)
An rvalue represents a temporary value or object that is about to be destroyed. They don't have permanent memory address that we can access with the `&` operator. Consider the following:
```cpp
std::string getName() {return "Temporary";}

int main()
{
    //1. Literals (except string literals) are rvalues
    int x{42};                  //42 is an rvalue
    bool flag{true};            //true is an rvalue

    //2. Expressions returning temporary values are rvalues
    int y{x+10};                //`x+10` is an rvalue

    //3. Return values of functions (by value) are prvalues
    std::string s{getName()};   //getName() returns an rvalue

    //4. Objects marked with std::move are xvalues
    //This tells the compiler: "I am done with `s`, feel free to steal its resources"
    std::string s2{std::move(s)};//std::move(s) is an xvalue

    //5. Lambda expressions are rvalues
    auto lambda{[]{return 0;}}; //The lambda itself is a prvalue

    // --- What you CANNOT do with rvalues
    // int* p = &42;        // ERROR: Cannot take the address of an rvalue
    // (x+1) = 10;          // ERROR: x + 1 is a prvalue int, not assignable to a prvalue
    
    return 0;
}
```
## Beyong lvalues and rvalues
In the example above you may have noticed that we are referring to something more than just rvalues. What happens is that in C++ values have different categories depending on how they are used.
### prvalue (Pure rvalue)
Represents an expression that initializes an object. They don't have names or a persistent memory location. Because they are generally short lived, we are allowed to "move" their resources instead of performing copying operations on them. `45`, `true` or `nullptr` are examples of prvalues. Consider the following:
```cpp
int x{10 + 5};
```
In the tiny example above, `10+5` will result in the value `15`, this value has no identity and no memory address assigned to it before it is stored in `x`. In other words, prvalues are the starting point for values before they are stored in named variables.
### xvalue (eXpiring Value)
An xvalue is a real object in memory, something that can be pointed to that the compiler also treats as movable. We don't write one directly; we produce it by casting an lvalue to an rvalue reference, which is exactly what `std::move` does.
```cpp
std::string s1{"Data"};
std::string s2{std::move(s1)};   // std::move(s1) is an xvalue
```
`std::move(s1)` takes the lvalue s1 and yields an xvalue: the same object, now given permission to be moved from.
### glvalue (Generalized lvalue)
These are usually considered a hybrid between lvalues and xvalues as glvalues are both expressions that have names and addresses and expiring values that have identity, but can be marked as movable. Consider the following:
```cpp
int a{5};
a = 10; //`a` is a glvalue
```
`a` can be identified by name and occupies a specific memory location. In the current example, `a` is used as an lvalue, but if we call `std::move(a)`, then the expression would be a glvalue because it would also be an xvalue.
![value categories](/img/blog/valuecategories.jpg)
## rvalue References
This is a type of reference that identifies objects that can be "moved". They can be used to extend the lifetime of temporary objects that are about to be destroyed. We use them to define move constructors or move assignment operators. You will notice that rvalue references are declared like this `T&&` and they will only be bound to rvalues, such as literals or temporary objects as we had already explained.
Consider the following:
```cpp
#include <iostream>
#include <string>

std::string getName() { return "Temporary"; }

int main()
{
    int a{2};
    int b{3};

    int&& x{42};                    // binds to a temporary holding the literal 42
    int&& sum{a + b};               // binds to the temporary result of a + b
    std::string&& name{getName()};  // binds to the temporary returned by value

    // int&& bad{a};                // ERROR: 'a' is an lvalue; T&& won't bind to an lvalue

    std::cout << x << ' ' << sum << ' ' << name << '\n';   // 42 5 Temporary
    return 0;
}
```
In real life, rvalue references are mostly used as function parameters, so that we can take advantage of move semantics. Consider the following:
```cpp
void push_back(T&& value);  //Accepts an rvalue reference to move the data
```
# Adding push_back(T&&)
With the basic explanations out of the way, we can now try and put them in practice by adding a new function to our `RingBuffer`. Consider the following:
```cpp {hl_lines=[7]}
template <typename T, std::size_t N>
class RingBuffer
{
public:
    //Previous code remains the same
    void push_back(const T& new_element);   //Adds a new element into the buffer
    void push_back(T&& new_element);        //Adds new element using an rvalue reference
    //The rest of the code remains the same

private:
    T* slot(std::size_t i) {
        return reinterpret_cast<T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    const T* slot(std::size_t i) const {
        return reinterpret_cast<const T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    alignas(T) std::byte buffer_[N * sizeof(T)];
    std::size_t head_;                         
    std::size_t tail_;                         
    std::size_t count_;                
};
```
And you will notice that not much changed in the definition, consider the following:
```cpp {hl_lines=[2, 10]}
template <typename T, std::size_t N>
void RingBuffer<T, N>::push_back(T&& new_element) { //Passing an rvalue reference
    if (count_ == N) {
        slot(0)->~T();
        head_ = (head_ + 1) % N;
    }
    else {
        ++count_;
    }
    new (slot(count_ - 1)) T(new_element); // first attempt: copies (new_element is a named lvalue here)
    tail_ = (tail_ + 1) % N;
}
```
However after running the original example, we find that the results are the same:
```
Pushing Temporary:
 [Ctro] Created 1
 [COPY] Duplicating 1 (4000 bytes)
 [Dtor] Destroyed 1
Done
 [Dtor] Destroyed 1
```
Remember that in C++ any named object is by default treated as an lvalue. While we are declaring `new_element` as an rvalue reference, we are using it as an lvalue, so the compiler defaults to copying instead of "moving". We fix this by using `std::move()`, consider the following:
```cpp {hl_lines=[11, 12]}
template <typename T, std::size_t N>
void RingBuffer<T, N>::push_back(T&& new_element) { //Passing an rvalue reference
    if (count_ == N) {
        slot(0)->~T();
        head_ = (head_ + 1) % N;
    }
    else {
        ++count_;
    }
    // std::move() casts the lvalue back to an rvalue to trigger the move constructor
    new (slot(count_ - 1)) T(std::move(new_element));
    tail_ = (tail_ + 1) % N;
}
```
But then running the example again, we find that nothing really has changed:
```
Pushing Temporary:
 [Ctro] Created 1
 [COPY] Duplicating 1 (4000 bytes)
 [Dtor] Destroyed 1
Done
 [Dtor] Destroyed 1
```
So, what is happening? Well, if we look at the `LoudHeavy` definition, we will find that it does not have a move constructor. Normally, the compiler will auto generate constructors for us, but since we created a custom copy constructor, it is now up to us to create a move constructor as well. Consider the following:
```cpp {hl_lines=["10-15"]}
struct LoudHeavy {
    int id;
    int* data;
    static constexpr std::size_t data_size{ 1000 };

    LoudHeavy(int i) : id{ i }, data{ new int[data_size] } {
        std::cout << " [Ctro] Created " << id << '\n';
    }

    // Move constructor: Steals the pointer instead of allocating new memory
    LoudHeavy(LoudHeavy&& other): id{ other.id }, data{other.data} {
        other.data = nullptr; //Reset the source, so that its destructor does nothing
        other.id = 0;
        std::cout << " [MOVE] Stealing data from " << id << '\n';
    }

    LoudHeavy(const LoudHeavy& other) : id(other.id), data(new int[data_size]) {
        std::memcpy(data, other.data, data_size * sizeof(int));
        std::cout << " [COPY] Duplicating " << id << " (" << data_size * sizeof(int) << " bytes)\n";
    }

    ~LoudHeavy() {
        delete[] data;
        std::cout << " [Dtor] Destroyed " << id << "\n";
    }
};
```
With this change, we can run the original example again and we get the following:
```
Pushing Temporary:
 [Ctro] Created 1
 [MOVE] Stealing data from 1
 [Dtor] Destroyed 0
Done
 [Dtor] Destroyed 1
```
Success! We have now successfully moved resources from a source into a target, but there is still a question. What really happens to the source object? What does it mean when we say valid, but unspefied state?
## Valid But Unspecified
As we know, moved-from-state happens when an object's resources have been transferred to another object via move constructor or move assignment. The source object, in other words, the object we moved the resources from remains valid, this means that the object still exists and can still be used without causing any undefined behavior, but we say it is unspecified because although the object is still there, its members MIGHT have been zeroed or in the case of `std::string` or `std::vector` they would be left empty. Notice that we say MIGHT because this is implementation dependent.
The object itself can still be assigned a new value.
Finally, here is the buffer code after all the updates:
```cpp
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

        // Use slot() to get the reinterpreted address and derefence it
        T& operator*() const {
            return *rb_->slot(index_);
        }

        // operator-> should return the pointer from slot() directly
        pointer operator->() const {
            return rb_->slot(index_);
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
            return *rb_->slot(index_);
        }

        pointer operator->() const {
            return rb_->slot(index_);
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
    void push_back(T&& new_element);        //Adds new element using an rvalue reference
    void pop_front();                       //Deletes the oldest element from the buffer
    bool empty() const;                     //Is the buffer empty
    bool full() const;                      //Is the buffer full
    const T& front() const;                 //Retrieves the oldest element
    std::size_t size() const;               //Retrieves the number of elements in the buffer

private:
    T* slot(std::size_t i) {
        return reinterpret_cast<T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    const T* slot(std::size_t i) const {
        return reinterpret_cast<const T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    alignas(T) std::byte buffer_[N * sizeof(T)];    //Usually changed to raw bytes for placement new
    std::size_t head_;                              //Index of oldest element (read position)
    std::size_t tail_;                              //Index of next write position
    std::size_t count_;                             //Total number of elements in the buffer
};

//Now, everywhere where we had the harcoded 8 can be update to use N

template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer() : head_{}, tail_{}, count_{} {}

template <typename T, std::size_t N>
RingBuffer<T, N>::~RingBuffer() 
{
    //Manually destroy only the active objects
    for (std::size_t i = 0; i < count_; ++i) {
        slot(i)->~T();
    }
}

template <typename T, std::size_t N>
void RingBuffer<T, N>::print_buffer() const
{
    for (const auto& element : *this) {
        std::cout << element << ' ';
    }
    std::cout << '\n';
}

template <typename T, std::size_t N>
void RingBuffer<T, N>::push_back(const T& new_element)
{
    if (count_ == N) {
        //1. Explicitly destroy the oldest element to release its resources
        slot(0)->~T();

        //2. Advance the head to the next logical position
        head_ = (head_ + 1) % N;
    }
    else {
        //Increment countif we are not ovewriting
        ++count_;
    }

    //3. Construct the new object in the Logical "end" slot using placement new
    new (slot(count_ - 1)) T(new_element);

    //4. Update the physical tail tracker for consistency
    tail_ = (tail_ + 1) % N;
}

template <typename T, std::size_t N>
void RingBuffer<T, N>::push_back(T&& new_element) {
    if (count_ == N) {
        slot(0)->~T();
        head_ = (head_ + 1) % N;
    }
    else {
        ++count_;
    }
    new (slot(count_ - 1)) T(std::move(new_element));
    tail_ = (tail_ + 1) % N;
}

template <typename T, std::size_t N>
void RingBuffer<T, N>::pop_front()
{
    if (count_ == 0) return;

    //Manually end the lifetime of the oldest element
    slot(0)->~T();

    head_ = (head_ + 1) % N;
    --count_;
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
    return *slot(0);
}

template <typename T, std::size_t N>
std::size_t RingBuffer<T, N>::size() const
{
    return count_;
}
```
## pass-by-value + std::move
We can combine pass by value with `std::move`. This is an alternative that allows us to avoid boilerplate of maintaining multiple overloads. This will allow us to have a single, unified function, but it will come with the cost of an extra move operation for every call. An lvalue argument will result in one copy and one move, while an rvalue will result in two moves. Consider the following:
```cpp
template<typnename T, std::size_t N>
void RingBuffer<T, N>::push_back(T new_element){ //Accepted by value
    if(count_ == N){
        slot(0)->~T();
        head_ = (head_ + 1) % N;
    }else{
        ++count;
    }
    //Move the local 'new_element' into the buffer
    new (slot(count_ -1)) T(std::move(new_element));
    tail_ = (tail_ + 1) % N;
}
```
In the code above, if an lvalue is passed, the argument is copied into the parameter `new_element`, then moved into the buffer, if an rvalue is passed, the argument is then moved into `new_element` and then moved again into the buffer. While this would work for our `RingBuffer`, we deliberately chose to make two different overloads of the same method, we did this to avoid the cost of an extra move operation.
## What We Accomplished
At the beginning of this post we had a `RingBuffer` that copied every element into storage, even when the element was a temporary about to be destroyed. We came to the conclusion that this was wasteful and fixed it by defining a second overload `push_back` method that can move things around. We also talked about value categories (lvalue, rvalue and the leaf categories) and added a move constructor to our `LoudHeavy` class. Now our `RingBuffer` moves rvalues into storage instead of copying them.

## What's Next?
While we got rid of a wasteful copy, there are some other things we need to take care of, look at how we use `push_back()`
```cpp
rb.push_back(LoudHeavy{1});
```
In this code we are building a temporary `LoudHeavy{1}`, then move it into the slot. The copy is gone, yes, but the temporary and themove are still there, one construction on the stack and one move into the buffer. In Part 8 we will remove that with `emplace_back()` which constructs the element directly in the slot. We will also look into variadic templates and perfect forwarding.
But that's not all! Our `RingBuffer` still does not have a propr copy and move constructors, so we will explore the rule of 5, then we will talk about reverse iterators and then we will implement a few utility methods like `back()`, `clear()` and `capacity()`.

# References / Sources

- [Value categories](https://en.cppreference.com/w/cpp/language/value_category)
- [Reference declaration (lvalue & rvalue references)](https://en.cppreference.com/w/cpp/language/reference)
- [std::move](https://en.cppreference.com/w/cpp/utility/move)
- [Copy constructors](https://en.cppreference.com/w/cpp/language/copy_constructor)
- [Move constructors](https://en.cppreference.com/w/cpp/language/move_constructor)