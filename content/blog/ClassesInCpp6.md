+++
title = "Ring Buffer Series Part 6 — Storage and Lifetimes"
date = "2026-05-15T12:00:42+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "classes"]

[images]
    featured_image = "/img/blog/ringbuffer6.jpg"
+++

![class](/img/blog/ringbuffer6.jpg)
We closed [part 5](ClassesInCpp5) by promising to tackle move semantics, but there is foundational work that needs to be done before that can happen. Move semantics will come in the future, but in this post we need to make sure our storage is capable of supporting it and we will have to rebuild one of the components we made in [part 2](ClassesInCpp2), so buckle up because this will be a long one.

Our current `RingBuffer` implementation relies on a fixed-size array (`T buffer_[N]`), which is simple. The issue with this approach is that the buffer is treated as an array of objects that already exist from the moment the buffer is created and this leads to three fundamental flaws.
![Buffer Storage Comparison](/img/blog/BufferStorageComparison.jpg)

1. ### Wasted Default Construction
The buffer is declared as an array of type `T`, because of this C++ has no choice but to default-construct all `N` elements the moment the `RingBuffer` is instantiated, this is ok for simple types, like `int`, but it has a significant performance cost when using `std::string` or a custom type. We are paying for creating objects that may never be used and on top of that if our custom type does not have a default constructor, our code will fail to compile. Consider the following:
```cpp
struct Tracer
{
    Tracer() {std::cout << "Default Constructed\n";}
};

int main()
{
    //Creating a buffer of 5 elements
    RingBuffer<Tracer, 5> buffer;
    //OUTPUT: "Default Constructed" is printed 5 times immediately
    //even though the buffer is logically empty
    return 0;
}
```
The output will be:
```
Default Constructed
Default Constructed
Default Constructed
Default Constructed
Default Constructed
```

2. ### Assignment vs Construction in push_back
The array slots are filled with default-constructed objects, so our `push_back` definition cannot "create" a new object in the buffer, it must use assignment (`operator=`) to overwrite an already existing object. This is less efficient than direct construction and is not the correct way of adding a "new" element into a container. Consider the following:
```cpp
struct Tracer
{
    Tracer() = default;
    Tracer& operator=(const Tracer&){
        std::cout << "Assigned\n";
        return *this;
    }
};

int main()
{
    //Creating a buffer of 5 elements
    RingBuffer<Tracer, 5> buffer;
    buffer.push_back(Tracer{});
    //OUTPUT: "Assigned"
    //The "new" element is actually just an old object getting a new value.
    return 0;
}
```
The output will be:
```
Assigned
```

3. ### "Ghost" Objects in pop_front
In our implementation of `pop_front` we are only updating internal bookkeeping (`head_` or `count_`), but it does not actually destroy the object being removed. The object remains in the buffer, holding onto its memory or resources until it is overwritten by a future `push_back`. This is a problem and it can be dangerous if `T` happens to manage resources like file handles. Consider:
```cpp
struct Tracer
{
    ~Tracer(){
        std::cout << "Destructor called\n";
    }
};

int main()
{
    //Creating a buffer of 5 elements
    RingBuffer<Tracer, 5> buffer;
    buffer.push_back(Tracer{});
    buffer.push_back(Tracer{});
    buffer.pop_front();
    std::cout << "Popped from buffer\n";
    //OUTPUT: "Popped from buffer"
    //Notice "Destructor Called" is not printed yet
    //The object still lives in the array slot
    return 0;
}
```
Output will be:
```
Destructor called    ← first push_back temporary dies
Destructor called    ← second push_back temporary dies
Popped from buffer
Destructor called    ← buffer destruction at end of main, slot 0
Destructor called    ← slot 1
Destructor called    ← slot 2
Destructor called    ← slot 3
Destructor called    ← slot 4
```
Notice that there are a bunch of destructor calls before and after `pop_front`, but none of them happens at `pop_front`.
Let's recap the problems by putting everything together, consider the following:
```cpp
struct Loud
{
    Loud() {std::cout << " [Loud] Default Constructed\n";}
    ~Loud() {std::cout << " [Loud] Destroyed\n";}
};
```
Now, watch what happens when we create a buffer that has 4 elements, but we only use 2.
```cpp
int main()
{
    std::cout << "--- Step 1: Instantiate RingBuffer<Loud, 4> ---\n";
    RingBuffer<Loud, 4> buffer;
    
    std::cout << "\n--- Step 2: Push Two Elements ---\n";
    buffer.push_back(Loud{});
    buffer.push_back(Loud{});

    std::cout << "\n--- Step 3: End of main() ---\n";
    return 0;
}
```
Here is the output with explanatory comments:
```
--- Step 1: Instantiate RingBuffer<Loud, 4> ---
 [Loud] Default Constructed
 [Loud] Default Constructed
 [Loud] Default Constructed
 [Loud] Default Constructed

--- Step 2: Push Two Elements ---
 [Loud] Default Constructed     //Temporary created for push_back
 [Loud] Destroyed               //Temporary destroyed after assignment
 [Loud] Default Constructed     //Temporary created for push_back
 [Loud] Destroyed               //Temporary destroyed after assignment

--- Step 3: End of main() ---
 [Loud] Destroyed               //Buffer element 0
 [Loud] Destroyed               //Buffer element 1
 [Loud] Destroyed               //Buffer element 2 (never used!)
 [Loud] Destroyed               //Buffer element 3 (never used!)
```
In Step 1 the buffer default-constructed all 4 slots before we even called `push_back` once. If we only ever stored one item, we paid for 2 "zombie" objects that did nothing but take time and resources. Also, notice that `push_back` did not "create" the element in the buffer, it assigned a value to one that was already there, which is why we see the temporary object being constructed and immediately destroyed.

## What We Actually Want
What we want is to replace the automatic, wasteful construction with precise manual control over the object lifetimes. The idea is to make sure that resources are used only when an element is logically present. Here is the process:

### 1. Raw Bytes (Allocation Without Construction)
We begin with a block of memory that is allocated but uninitialized. Rather than using an array such as `T buffer_[N]` which forces immediate default construction, we use raw memory storage with `std::byte` array. At this point the memory contains "garbage" data and the compiler does not recognize any objects as living there.

### 2. Object Constructed When Pushed
When a new element is added via `push_back` an in-place construction is performed. We use placement new syntax `new (address) T(args)` which invokes the constructor at the provided memory location without requesting new storage from the heap, this transforms the raw bytes into a fully initialized object, thus beginning its lifetime.

### 3. Object Destroyed When Popped or Overwritten
When an element is removed via `pop_front` or replaced, its lifetime must be ended explicitly. We are not able to use the `delete` operator because the container owns the memory separately from the object, if we use `delete` that would try to free the memory block itself. We must perform manual destruction by calling the object's destructor directly `ptr->~T()`. This will ensure that the resources the object held are released when the element is removed from the container.

### 4 Return to Raw Bytes
When the destructor completes, it means the object does not exist anymore and its slot in the container reverts back to raw, uninitialized bytes. The underlying memory remains allocated and is kept in reserve by the container. This slot is ready to house a new object in a future push operation without having to create a "zombie" object.

![lifecycle](/img/blog/lifecycle.jpg)

## Uninitialized Storage: alignas(T) std::byte buffer_[N * sizeof(T)]
Before we start making any changes, we must familiarize ourselves with a few more concepts:
### std::byte
`std::byte` is a type designed to represent raw memory as a collection of bits. This type differentiates itself from others like `char` or `int` in that we can't perform mathematical operations like addition or subtraction on it. This is so that we are protected from treating raw memory as a number. The main purpose of `std::byte` is to represent bit patterns, so it explicitly supports bitwise logical operators (`&`,`|`,`^`,`~`,`<<`, and `>>`) and it occupies exactly one byte (on virtually every platform you will encounter, that's 8 bits). `std::byte*` can alias any address in memory which makes it safe tool for low level byte manipulation. In modern C++ `std::byte` is the preferred way to indicate that we are working with raw bytes rather than with characters.
### alignas(T)
We use `alignas(T)` to tell the compiler that the memory address for our raw storage starts on a boundary that is valid for type `T`. In the case of an `int`, the storage must start at an address divisible by 4 or 8. We need this because we are switching from an array of objects `T buffer_[N]` to raw bytes `std::byte buffer_[S]` and this change will cause the compiler to lose the type information. A `std::byte` array has an alignment of 1, this means that it can start anywhere, not only that, but we also risk crashing on some CPUs if we treat a misaligned address as a complex type and it would even be a problem with CPUs where this works as those CPUs will have to perform extra operations to piece our object together, so, even if it doesn't crash, it will be painfully slow. We use `alignas(T)` to guarantee that our raw bytes are a safe space to construct an object of type `T` during placement new.

### Why [N * sizeof(T)]
The `sizeof` operator is designed to take care of contiguous storage requirements. This operator includes the sum of the type's data members and also any trailing padding added by the compiler. This trailing padding makes sure that if the first object in a sequence is properly aligned, every subsequent object will also land on a valid alignment boundary. The specific `[N * sizeof(T)]` works because in C++ arrays and container buffers for things like `std::vector` store elements one after another with no memory gaps between them, so multiplying the total size of one object by `N` gives us the exact space needed for the entire sequence.

## Placement New
Written as `new (location) T(args...)`, placement new is a version of the `new` operator that constructs an object in a pre-allocated memory area instead of requesting new memory from the heap. The standard `new` performs two distinct actions: it allocates enough raw bytes for the type and then constructs the object by calling its constructor. Placement new does not perform the allocation step, only the construction, this allows us to put an object onto a specific address, such as a buffer of raw bytes or a memory mapped hardware register. When we use `new (location) T(args...)` the compiler looks for an overload of `operator new` that takes `void*` as an additional argument, the one provided by the standard library looks like this:
```cpp
void* operator new(std::size_t size, void* location) noexcept
{
    return location; //This returns the address that we provided it.
}
```
After this, the compiler invokes the constructor of type `T` at the returned address.
As mentioned, placement new does not manage allocation, as a consequence we are not able to use the standard `delete` operator. `delete` would attempt to return memory to the heap, this would lead to corruption since it is likely that the memory we are using comes from a local buffer. To address this, we need to call the object's destructor directly `ptr->~T()`. Consider the following:
```cpp
#include <new>
#include <iostream>

struct Loud
{
    int id;
    Loud(int i) : id{i} {std::cout << "Constructed " << id << '\n';}
    ~Loud(){std::cout << "Destructed " << id << '\n';}
};

int main()
{
    //1. Pre-allocated raw memory (properly aligned for Loud)
    alignas(Loud) std::byte buffer[sizeof(Loud)];

    //2. Construct Loud in that specific buffer
    Loud* l_ptr{new (buffer) Loud(42)};

    //3. Use the object
    std::cout << "Loud ID: " << l_ptr->id << '\n';

    //4. Manually destroy the object (DO NOT USE DELETE)
    l_ptr->~Loud();
    return 0;
}
```
The output for this code is:
```
Constructed 42
Loud ID: 42
Destructed 42
```
## Explicit Destructor Calls
The expression `p->~T()` allows us to manually end the lifetime of an object without releasing the underlying memory it occupies. Normally when using `delete` two things happen, the object's destructor is called and then the memory is deallocated. `p->~T()` only does the first step, so the object gets turned back into raw bytes. This syntax is required after placement new has been used to construct a new object, remember that placement new does not allocate any memory, so if we used a standard `delete` the program would crash because it is trying to free memory it does not own. We will be using this in our RingBuffer, so that when we "pop" an element, we call its destructor, but we keep the slot available for a future push, no need to allocate that memory again. Consider the following:
```cpp
#include <new>
#include <iostream>

struct Loud
{
    ~Loud(){std::cout << "Destructor Called!\n";}
};

int main()
{
    //1. Allocate raw memory for one Loud object
    alignas(Loud) std::byte buffer[sizeof(Loud)];
    Loud* p{reinterpret_cast<Loud*>(buffer)};

    //2. Construct the object in place
    new (p) Loud();

    //3. Manually destroy the object
    p->~Loud();

    //At this point, "buffer" still exists as raw bytes,
    //but the Loud object is officially dead.
    return 0;
}
```
Output:
```
Destructor Called!
```
You might have noticed the `reinterpret_cast` in our code. It will all be explained in a few moments.
## What is Casting?
In programming, casting is the process of converting a value from one type to another. In C++ we have four kinds of casting available to us that are intended to document programmer's intent, so that we don't rely on the more dangerous C-Style cast.
### static_cast
This is the most common casting operator in C++. It performs conversions that are well defined by the C++ language. It does checks at compile time which means that there is zero runtime overhead. We use `static_cast` for numerical conversions between basic types, such as `float` to `int` or `int` to `double` acknowledging potential data loss. We also can use it for class hierarchy navigation by either converting a pointer or reference of a derived class to a base class or converting a base pointer or reference into a derived type.
```cpp
#include <iostream>

struct Base{virtual ~Base() = default;};
struct Derived : Base {void action() {std::cout << "Derived Action!\n";}};

int main()
{
    //1. Numeric Conversion
    double pi{3.14159};
    int truncated{static_cast<int>(pi)};
    std::cout << "Double: " << pi << " -> Int: " << truncated << '\n';

    //2. Downcasting (Safe here because we know 'b' points to a `Derived`)
    Base* b{new Derived()};
    Derived* d{static_cast<Derived*>(b)};
    d->action();

    //3. Void pointer conversion
    void* raw{&truncated};
    int* pInt{static_cast<int*>(raw)};
    std::cout << "Value via void*: " << *pInt << "\n";

    delete b;
    return 0;
}
```
Output:
```
Double: 3.14159 -> Int: 3
Derived Action!
Value via void*: 3
```
### dynamic_cast
This is a runtime mechanism that is available to us for safe navigation within a class hierarchy. `dynamic_cast` performs runtime checks to verify that the conversion we are trying to perform is valid. We use it for converting base class pointer or reference to a derived class. It also allows us to ask an object if it supports a specific interface at runtime. This kind of casting incurs a performance cost because of the runtime checks.
```cpp
#include <iostream>
#include <typeinfo> // For std::bad_cast

struct Base{virtual ~Base() = default;};
struct Derived : Base{void action(){std::cout << "Derived Action!\n";}};
struct Other : Base{};

int main()
{
    Base* b1{new Derived()};
    Base* b2{new Other()};

    //1. Successful Pointer Downcast
    if(Derived* d{dynamic_cast<Derived*>(b1)}){
        std::cout << "b1 is a Derived object: ";
        d->action();
    }

    //2. Failed Pointer Downcast
    if(Derived* d{dynamic_cast<Derived*>(b2)}){
        d->action();
    }else{
        std::cout << "b2 is NOT a Derived object (Returned nullptr)\n";
    }

    //3. Failed Reference Downcast
    try{
        Base& ref{*b2};
        Derived& d_ref{dynamic_cast<Derived&>(ref)};
    }catch(const std::bad_cast& e){
        std::cout << "Reference cast failed: " << e.what() << '\n';
    }

    delete b1;
    delete b2;
    return 0;
}
```
Output:
```
b1 is a Derived object: Derived Action!
b2 is NOT a Derived object (Returned nullptr)
Reference cast failed: std::bad_cast
```
### const_cast
This is the only C++ operator that can remove or add `const` from a variable. It doesn't deal with objects but with pointers and references. We use `const_cast` to interface with legacy code where a function may take a non const variable `char*` for reading and we try to pass in a `const char*`, we can use `const_cast` to remove the `const` from our variable, so that it works on that specific function call. It can also be useful to avoid code duplication via Scott Meyers Pattern (explained in a future post) and for low level memory management. `const_cast` should never be used to cast away `const` from an object that was created as `const`.
```cpp
#include <iostream>

//Imagine this is a legacy function that we cannot change
//It takes a non-const pointer but only reads the data
void legacy_print(int* p)
{
    std::cout << "Legacy Print: " << *p << '\n';
}

int main()
{
    const int value{100};
    //legacy_print(&value) //Error: cannot convert const int* to int*

    //Correct use of const_cast to interface with the legacy API
    legacy_print(const_cast<int*>(&value));

    return 0;
}
```
Output:
```
Legacy Print: 100
```
### reinterpret_cast
This is the lowest level casting operator, people refer to it as the semantic sledgehammer because it forces the compiler to treat a bit pattern as a completely different type. As the name implies, it reinterprets raw bits by taking an underlying bit pattern of an expression and mapping it into a new type without changing data. It is used to convert between pointers or references of totally unrelated types, `int*` to `float*`, operations that are normally forbidden.
```cpp
#include <iostream>
#include <iomanip>

int main()
{
    float f{3.14f};

    //Treat the address of float `f` as an int pointer
    int* pInt{reinterpret_cast<int*>(&f)};

    std::cout << "Float value: " << f << '\n';
    std::cout << "Memory address: " << &f << '\n';

    //Display the bits as a decimal integer
    std::cout << "Bit pattern as int: " << *pInt << '\n';

    //Display the bits as a hexadecimal
    std::cout << "Bit pattern as hex: 0x" << std::hex << *pInt << '\n';
    return 0;
}
```
Output:
```
Float value: 3.14
Memory address: 0x7ffcbc2dde54
Bit pattern as int: 1078523331
Bit pattern as hex: 0x4048f5c3
```
This example works on most compilers, but it technically violates strict aliasing. Modern C++ provides `std::bit_cast<int>(f)` (C++20) and `std::memcpy(&i, &f, sizeof(i))` for legal bit-pattern reinterpretation. For our buffer, we'll only `reinterpret_cast` between `std::byte*` and `T*`. `std::byte` is allowed to alias any type, which is exactly why we chose it as our storage.

Casting in itself is a topic that will require its own series which I am already planning, but I felt it was important that we touched on the basics. In our specific case we are interested in `reinterpret_cast`.
## RingBuffer and reinterpret_cast
Recall that we are using `std::byte` array for our ring buffer, this means the compiler doesn't know what objects we are storing, so we can use `reinterpret_cast` as a bridge that lets the compiler know to treat those raw bits as a specific type `T` opening the door for operations that would normally be forbidden to us, for example after converting the raw bytes into an address of `T*` is what allows placement new to put an object into that memory, which in turn allows us to use `p->~T()` to destroy the object. We will make use of this by implementing a new function in our RingBuffer. We will put it in the private section of our `RingBuffer` class
```cpp
    T* slot(std::size_t i) {
        return reinterpret_cast<T*>(&buffer_[(head_ + i) % N]);
    }
    const T* slot(std::size_t i) const {
        return reinterpret_cast<const T*>(&buffer_[(head_ + i)] % N);
    }
    alignas(T) std::byte buffer_[N * sizeof(T)]; //Usually changed to raw bytes for placement new
    std::size_t head_;                 
    std::size_t tail_;                 
    std::size_t count_;                
```
With these changes in the class, we can now update our `push_back` and `pop_front` methods.
### Updated push_back
Recall that in the original, we used `buffer_[tail_] = new_element;`. Our new `slot()` method will replace that assignment with placement new to construct the object directly into raw memory:
![updated_push_back1](/img/blog/updated_push_back1.jpg)
![updated_push_back2](/img/blog/updated_push_back2.jpg)
![updated_push_back3](/img/blog/updated_push_back3.jpg)
```cpp
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
        //Increment count if we are not overwriting
        ++count_;
    }

    //3. Construct the new object in the Logical "end" slot using placement new
    new (slot(count_ - 1)) T(new_element);

    //4. Update the physical tail tracker for consistency
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
```
Because we are now using raw bytes and placement new, the constructor needs to change as well.
Notice that we purposely do not initialize `buffer_`. This omission is the entire architectural change. The storage exists, but no objects live in it yet. The constructor only has to worry about zero initialize the bookkeeping variables that will track which slots get brought to life later.
```cpp
template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer() : head_{}, tail_{}, count_{} 
{}
```
The `front()` function also needs to be updated because as it currently is, it will try to return a reference to a single `std::byte` and not the oldest `T&` element in the container.
```cpp
template <typename T, std::size_t N>
const T& RingBuffer<T, N>::front() const
{
    assert(!empty());
    return *slot(0);
}
```
The most important update happens with the destructor. Our buffer consists of raw bytes, so the compiler will not automatically call the destructors for any `T` objects remaining in the buffer when it goes out of scope, we have to manually iterate through all active elements and call their destructors ourselves.
```cpp
template <typename T, std::size_t N>
RingBuffer<T, N>::~RingBuffer() 
{
    //Manually destroy only the active objects
    for (std::size_t i = 0; i < count_; ++i) {
        slot(i)->~T();
    }
}
```
We also must ensure that our iterator returns references to our `T` objects and not to a raw `std::byte`. We achieve this by updating both `iterator` and `const_iterator`.
### iterator
We add the following code:
```cpp
        // Use slot() to get the reinterpreted address and derefence it
        T& operator*() const {
            return *rb_->slot(index_);
        }

        // operator-> should return the pointer from slot() directly
        pointer operator->() const {
            return rb_->slot(index_);
        }
```
### const_iterator
This class has similar changes:
```cpp
        //Returns const reference
        const T& operator*() const {
            return *rb_->slot(index_);
        }

        pointer operator->() const {
            return rb_->slot(index_);
        }
```
We use the `slot(index_)` to hide complex physical address mapping and wraparound logic from the iterator, `const_iterator` calls the `const` version of `slot()` and we add the `operator->` to allow us to access members of `T` using `it->some_method()` syntax via the returned pointer from `slot()`, there is one benefit that all of this work already provides, consider the following:
```cpp
template <typename T, std::size_t N>
void RingBuffer<T, N>::print_buffer() const
{
    for (const auto& element : *this) {
        std::cout << element << ' ';
    }
    std::cout << '\n';
}
```
Since we went through the trouble of updating our iterators, the manual loop can be replaced with a range based `for` loop, simplifying the logic.
## Seeing it in Action
Let's try the current changes with the following program:
```cpp
#include "ringbuffer.h"
#include <string>

struct Loud
{
    std::string name;
    Loud(std::string n) : name{ std::move(n) } {
        std::cout << " Loud Constructor " << name << '\n';
    }
    ~Loud() { std::cout << " Loud Destructor " << name << '\n'; }

    //Support Printing
    friend std::ostream& operator<<(std::ostream& os, const Loud& l) {
        return os << l.name;
    }
};

int main()
{
    std::cout << "--- Initializing RingBuffer<Loud, 2> ---\n";
    RingBuffer<Loud, 2> rb;

    std::cout << "\nPushing A and B:\n";
    rb.push_back(Loud{ "A" });
    rb.push_back(Loud{ "B" });

    std::cout << "\nBuffer Contents: ";
    rb.print_buffer(); //Uses updated slot() logic

    std::cout << "\nPushing C (should overwrite A):\n";
    rb.push_back(Loud{ "C" });
    
    std::cout << "\nPopping front (removes B):\n";
    rb.pop_front();

    std::cout << "\nEnding main scope (Buffer destructor calls remaining destructors)...\n";
    return 0;
}
```
Here is the output:
```
--- Initializing RingBuffer<Loud, 2> ---

Pushing A and B:
 Loud Constructor A
 Loud Destructor A
 Loud Constructor B
 Loud Destructor B

Buffer Contents:
Program terminated with signal: SIGSEGV
```
Our program has crashed, but why is that? The problem lies in our implementation of the `slot()`.
Consider this code:
```cpp
T* slot(std::size_t i) {
    return reinterpret_cast<T*>(&buffer_[(head_ + i) % N]);
}
const T* slot(std::size_t i) const {
    return reinterpret_cast<const T*>(&buffer_[(head_ + i) % N]);
}
```
The problem is that because of the changes we made `buffer_` is now an array of raw `std::byte`, this means the each element in the container is separated by 1 byte and not by `sizeof(T)` bytes apart. In other words, the container "forgot" to account for whatever size `T` would be, let me explain. With `sizeof(Loud)` around 32 bytes on most implementation, `slot 0` points at byte 0 and `slot 1` points at byte 1, so they overlap by 31 bytes. When we call placement new to put `B` into `slot 1`, we stomp the first byte of `A`'s `std::string`. That byte holds part of the string's internal pointer or size word, so the next time `print_buffer` dereferences `slot 0`, we follow a garbage pointer and crash.
Here is the corrected code:
```cpp
T* slot(std::size_t i) {
    return reinterpret_cast<T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
}
const T* slot(std::size_t i) const {
    return reinterpret_cast<const T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
}
```
And when we run the example again this is the output:
```
--- Initializing RingBuffer<Loud, 2> ---

Pushing A and B:
 Loud Constructor A
 Loud Destructor A
 Loud Constructor B
 Loud Destructor B

Buffer Contents: A B 

Pushing C (should overwrite A):
 Loud Constructor C
 Loud Destructor A
 Loud Destructor C

Popping front (removes B):
 Loud Destructor B

Ending main scope (Buffer destructor calls remaining destructors)...
 Loud Destructor C
```
And with this our RingBuffer is almost complete.
## Default Copy, The Missing Pieces
Although the RingBuffer currently works for all intents and purposes, there is an issue that we introduced when we switched to the new storage and this is something the compiler will not warn us about and will not show up until someone uses the buffer in ways we haven't thought of. Consider the following:
```cpp
struct Tracker
{
    int* resource;
    Tracker(int v) : resource{new int{v}} {
        std::cout << "Tracker(" << *resource << ") allocated " << resource << '\n';
    }
    Tracker(const Tracker& other) : resource{new int{*other.resource}} {
        std::cout << "Tracker(" << *resource << ") copy-allocated " << resource << '\n';
    }
    ~Tracker() {
        std::cout << "Tracker freeing " << resource << '\n';
        delete resource;
    }
};

int main()
{
    {
    RingBuffer<Tracker, 10> rb1;
    rb1.push_back(Tracker{1});

    RingBuffer<Tracker, 10> rb2 = rb1; //This looks harmless
    }
    return 0;
}
```
In the constructor, `Tracker` is allocating an `int` on the heap which the destructor then frees. Each `Tracker` object owns its own allocation. We create a `RingBuffer` and push a `Tracker` into it, we create a second `RingBuffer` and initialize it with the first. The problem is that `RingBuffer` does not have a copy constructor, so we rely on the compiler to generate one for us, which it does, but here is what happens when we run this program.
```
free(): double free detected in tcache 2
Program terminated with signal: SIGSEGV
```
There two allocations, but three deallocations, this is because the same heap address gets passed to `delete` twice. This happens because the copy constructor that the compiler generated performs member wise copying, this means that `head_`, `tail_` and `count_` will get copied by value, which is fine for simple types, however, `buffer_` is an array of `std::byte` and arrays are copied element by element or in this case byte by byte. This means that in the program, when `rb2` is created, `rb2.buffer_` holds a byte for byte duplicate of `rb1.buffer_`, these bytes represent a fully constructed `Tracker` object. After the copy, `rb1` and `rb2` have a `Tracker` at `slot 0` and both trackers carry the same heap pointer. We will expand more into this in a future post as this has already been quite the journey, but for now we will solve the problem implementing a simple approach. We will delete the copy constructor and the assignment operator from our RingBuffer:
```cpp
RingBuffer(const RingBuffer&)            = delete;
RingBuffer(RingBuffer&&)                 = delete;   // && — move constructor
RingBuffer& operator=(const RingBuffer&) = delete;
RingBuffer& operator=(RingBuffer&&)      = delete;   // && — move assignment
```
Since these functions no longer exists in our class, now our little program will not compile and will instead give us an error message
```
error: use of deleted function 'RingBuffer<T, N>::RingBuffer(RingBuffer<T, N>&) [with T = Tracker; long unsigned int N = 10]'
```
As mentioned, we will work on copy and move construction in a future post, when we explore the rule of five. In the meantime, here is the updated class.
```cpp
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
    RingBuffer(const RingBuffer&)            = delete;
    RingBuffer(RingBuffer&&)                 = delete;   // && — move constructor
    RingBuffer& operator=(const RingBuffer&) = delete;
    RingBuffer& operator=(RingBuffer&&)      = delete;   // && — move assignment

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

    void print_buffer() const;       //Outputs the buffers contents to the screen
    void push_back(const T& new_element);   //Adds a new element into the buffer
    void pop_front();                //Deletes the oldest element from the buffer
    bool empty() const;              //Is the buffer empty
    bool full() const;               //Is the buffer full
    const T& front() const;                 //Retrieves the oldest element
    std::size_t size() const;          //Retrieves the number of elements in the buffer

private:
    T* slot(std::size_t i) {
        return reinterpret_cast<T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    const T* slot(std::size_t i) const {
        return reinterpret_cast<const T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    alignas(T) std::byte buffer_[N * sizeof(T)]; 
    std::size_t head_;                         //Index of oldest element (read position)
    std::size_t tail_;                         //Index of next write position
    std::size_t count_;                        //Total number of elements in the buffer
};

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
        //Increment count if we are not overwriting
        ++count_;
    }

    //3. Construct the new object in the Logical "end" slot using placement new
    new (slot(count_ - 1)) T(new_element);

    //4. Update the physical tail tracker for consistency
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
## Bonus: Types Without Default Constructors
With the old `T buffer_[N]` storage, every slot was default constructed when the buffer came into existence. That forced `T` to have a default constructor, even if we never used one. Consider the following:
```cpp
struct Point
{
    int x;
    int y;
    Point(int xi, int yi) : x{xi}, y{yi} {}

    friend std::ostream& operator<<(std::ostream& os, const Point& p) {
        return os << "(" << p.x << "," << p.y << ")";
    }
};
```

`Point` requires both coordinates at construction; it has no default constructor. With the old storage, trying to use it would fail to compile:

```cpp
RingBuffer<Point, 4> points;  // Error: no matching function for call to 'Point::Point()'
```

The compiler failed to generate the `RingBuffer` constructor because it could not default-construct the array of `Point` objects. This hidden requirement blocked a valid type, even though the original declaration never explicitly demanded it.

With the new storage, no `T` objects exist when the buffer is constructed; the byte array is just memory. The constraint vanishes:

```cpp
int main()
{
    RingBuffer<Point, 4> points;
    points.push_back(Point{1, 2});
    points.push_back(Point{3, 4});
    points.print_buffer();   // (1,2) (3,4)
    return 0;
}
```

This compiles and runs. `Point` only exists in slots we explicitly construct it into, so its lack of a default constructor is irrelevant to the buffer.

This isn't a feature we designed for, it fell out of the architecture. But it widens the set of types our buffer can hold to any class with a non-trivial constructor, which covers most useful classes.
## What We Accomplished

We started Part 6 with a `RingBuffer` that worked but wasted construction calls, performed assignment where it should have constructed, and left ghost objects in slots after pop. We rebuilt the storage from the ground up:

- Switched from `T buffer_[N]` to `alignas(T) std::byte buffer_[N * sizeof(T)]`, so slots start as raw memory and only become live objects when we explicitly construct them.
- Introduced placement new for in-place construction and `p->~T()` for in-place destruction, the two primitives that bracket every object's lifetime in our container.
- Built a `slot()` helper that translates logical indices into properly-aligned pointers, hiding the modular arithmetic and `sizeof(T)` scaling from the rest of the class.
- Updated `push_back`, `pop_front`, the constructor, the destructor, `front()`, and both iterator types to use the new storage model, exactly the methods that touch element lifetimes.
- Discovered along the way that switching to byte storage breaks the compiler-generated copy and move operators, and `= delete`d them as the correct interim contract.

Our `RingBuffer` is now precise about what exists in it and when. No waste, no ghosts, no silent byte-copy corruption.

## What's Next?

Part 7 picks up where Part 5's closing teaser left off: move semantics. Our `push_back` still takes its argument by `const T&`, which means every element is *copied* into the buffer. For an `int` that's OK; for a `std::string` it's a heap allocation we didn't need. We'll add an rvalue overload (and explain what an rvalue is) that lets us move temporaries into the buffer instead of copying them, and a small instrumented class will make the difference visible in the console.

But that will not be the end; we still need to talk about the following:
- Proper copy and move constructors for the `RingBuffer` itself (the rule of five).
- `emplace_back`, with variadic templates and perfect forwarding.
- Reverse iterators.
- Utility methods like `back()`, `clear()`, and `capacity()`.

The buffer in its current shape is correct, complete, and ready to support each of those additions. We've done the hardest architectural work in this series; what comes next is refinement.

# References / Sources

- [std::byte](https://en.cppreference.com/w/cpp/types/byte)
- [alignas specifier](https://en.cppreference.com/w/cpp/language/alignas)
- [Object lifetime](https://en.cppreference.com/w/cpp/language/lifetime)
- [new expression (including placement new)](https://en.cppreference.com/w/cpp/language/new)
- [Destructors](https://en.cppreference.com/w/cpp/language/destructor)
- [Explicit type conversion (the four casts)](https://en.cppreference.com/w/cpp/language/explicit_cast)
- [reinterpret_cast](https://en.cppreference.com/w/cpp/language/reinterpret_cast)
- [Deleted functions](https://en.cppreference.com/w/cpp/language/function)
- [std::bit_cast](https://en.cppreference.com/w/cpp/numeric/bit_cast)