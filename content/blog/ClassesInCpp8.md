+++
title = "Ring Buffer Series Part 8 — Building in Place - emplace_back, variadic templates & perfect forwarding"
date = "2026-06-09T12:00:42+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "classes"]

[images]
    featured_image = "/img/blog/ringbuffer8.jpg"
+++

![class](/img/blog/ringbuffer8.jpg)
<br>We closed [part 7](ClassesInCpp7) with some unfinished business, recall that `push_back(T&&)` still builds a temporary object and then moves it. Consider the output of our last example:
```
Pushing Temporary:
   [Ctro] Created 1
   [MOVE] Stealing data from 1
   [Dtor] Destroyed 1
```
This works, but as a wise man once said; "we are still not happy". What if we could get it to just `[Ctro] Created 1`. No temporary, no move?
# The Goal: Construct From Arguments, Not From a Finished Object
Our `push_back()` takes a fully constructed object `T`. Unlike `push_back()`, `emplace_back()` takes a variadic number of arguments which are perfectly forwarded to the constructor of the element type `T`. This allows the object to be built directly in place within the container's memory. Before we can move on to the implementation, there are some concepts that need to be explained.
## Perfect Forwarding
This is a technique that allows function templates to pass arguments to another function while preserving their original value category. Normally, when a function receives an argument, the parameter itself becomes an lvalue, if we didn't have perfect forwarding, passing that parameter to a second function would trigger a copy instead of a move, even if the argument was movable. Consider the following:
```cpp
#include <utility>
#include <iostream>

void target(int& x) {std::cout << "Lvalue\n";}
void target(int&& x){std::cout << "Rvalue\n";}

template<typename T>
void wrapper(T&& arg) //Forwarding reference
{
    target(std::forward<T>(arg)); //Conditional cast
}

int main()
{
    int v{42};                  
    wrapper(v);                 //Passes lvalue -> prints "Lvalue"
    wrapper(42);                //Passes rvalue -> prints "Rvalue"
    wrapper(std::move(v));      //Passes xvalue -> prints "Rvalue"
    return 0;
}
```
When we use `&&` with a template parameter that is being deduced in that very call, it creates a forwarding reference (also known as a universal reference, the term coined by Scott Meyers before the standard settled on a name) that can be bound to both lvalues and rvalues. Keep the "deduced in that very call" part in mind, we will come back to it, because not every `&&` on a template parameter qualifies and we already have a counterexample sitting in our own buffer. `std::forward<T>(arg)` is a conditional cast that converts the parameter to an rvalue only if the original argument was an rvalue. If we pass an lvalue, then `T` is deduced as `int&`, the parameter remains an lvalue reference and `std::forward` does nothing.
## Variadic Arguments
These allow any function to take any number of arguments of various types. Ever since C++11 we have what we call Variadic Templates, which provide a type-safe, compile-time alternative for handling variable arguments. Consider the following:
```cpp
#include <utility>
#include <iostream>

//A target function that expects specific value categories
void targetFunction(int& a, double&& b)
{
    std::cout << "Target called with lvalue " << a << " and rvalue " << b << '\n';
}

//A variadic template wrapper
template <typename Func, typename... Args>
void callWithLogging(Func f, Args&&... args)
{
    std::cout << "Log: Starting function call...\n";

    //std::forward preserves the original value category (lvalue vs rvalue)
    //The ellipsis (...) expands the parameter pack into a comma-separated list
    f(std::forward<Args>(args)...);

    std::cout << "Log: Function call complete.\n";
}

int main()
{
    int x{10};

    //The wrapper forwards 'x' as an lvalue and 3.14 as an rvalue
    callWithLogging(targetFunction, x, 3.14);

    return 0;
}
```
The `typename... Args` syntax allows the wrapper to accept any number of arguments of any type, `Args&&` is the forwarding reference which can be bound to any lvalues or rvalues and `std::forward` inside the wrapper performs the conditional cast that ensures `x` remains an lvalue and `3.14` remains an rvalue when they get passed to `targetFunction`.
## Parameter Packs
A parameter pack is a feature that allows templates to accept an arbitrary number of arguments. There are two types of parameter packs; a template parameter pack `typename... Ts` represents a sequence of zero or more types and a function parameter pack `Ts... args` which represents a sequence of zero or more values corresponding to those types. Consider the following:
```cpp
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) //Parameter pack declaration
{
    //Parameter pack expansion
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```
The declaration `typename... Args` creates a template parameter pack and the `Args&&... args` creates a function parameter pack. The ellipsis (three dots) in the expression `std::forward<Args>(args)...` unpacks the arguments into a comma-separated list. For example if we have an argument pack containing 1,2,3, passing it to a function `f(args...)` will be understood as `f(1,2,3)`. This is called pack expansion or pack unpacking. There is no magic to the `...`, it simply takes the *pattern* to its left and repeats it once for every element of the pack, so `std::forward<Args>(args)...` becomes `std::forward<A0>(a0), std::forward<A1>(a1), std::forward<A2>(a2)`. Consider the following:
```cpp
#include <iostream>

struct Base1
{
    Base1(int x){std::cout << "Base1: " << x << '\n';}
};

struct Base2
{
    Base2(int x){std::cout << "Base2: " << x << '\n';}
};

//Template parameter pack declaration
template<typename... Bases>
class Derived : public Bases... //Expansion in Base Specifier List
{
    public:
    //Expansion in Function Parameter List
    Derived(const Bases&... args) : Bases(args)... //Expansion in Member Initializer List
    {
    }
};

int main()
{
    //Unpacks into: class Derived : public Base1, public Base2
    Derived<Base1, Base2> d{10, 20};
    return 0;
}
```
The example above illustrates pack expansion in three different locations: the base class list, the constructor parameters and the member initializer list.

# The Two Faces of &&
Before we touch the buffer, we have to clear up something very important. Look at these two declarations side by side. We wrote the first one in [part 7](ClassesInCpp7) and we are about to write the second one:
```cpp
void push_back(T&& new_element);          //Part 7

template <typename... Args>
T& emplace_back(Args&&... args);          //Part 8
```
Both have `&&`. They are NOT the same thing.

In `push_back(T&& new_element)`, `T` is the **class** template parameter. By the time anyone calls `push_back`, the buffer has already been instantiated as, say, `RingBuffer<LoudHeavy, 4>`, which means `T` *is* `LoudHeavy`. It is fixed. No deduction happens at the call site, so `T&&` is plain `LoudHeavy&&`: a **true rvalue reference** that binds to rvalues only. This is exactly why in part 7 we needed to keep the second overload `push_back(const T&)` around for lvalues, one signature could not take both.

In `emplace_back(Args&&... args)`, `Args` is deduced fresh **on every call**. That per-call deduction is what makes `Args&&` a **forwarding reference**: it binds to lvalues and rvalues alike, and it records which one it received.

So here is the rule, stated precisely: a forwarding reference is the bare form `T&&` where `T` is a template parameter *deduced in that very call*. The common belief that "`&&` on a template parameter means rvalue reference" is false, and our own `push_back(T&&)` is the counterexample; its `T` is a template parameter, but it was deduced when the class was instantiated, not when the function is called.

| | `push_back(T&& x)` | `emplace_back(Args&&... a)` |
|---|---|---|
| Where the type comes from | Fixed when `RingBuffer<T,N>` was instantiated | Deduced per call |
| What `&&` means | True rvalue reference | Forwarding reference |
| What it binds to | Rvalues only | Lvalues AND rvalues |

## Reference Collapsing
"Binds to both" sounds like magic until you see the mechanism behind it. C++ does not allow a reference to a reference to exist directly, so when template deduction produces one, the compiler *collapses* it according to four rules:

| Combination | Collapses to |
|---|---|
| `T&` `&` | `T&` |
| `T&` `&&` | `T&` |
| `T&&` `&` | `T&` |
| `T&&` `&&` | `T&&` |

Only rvalue-reference-to-rvalue-reference survives as `&&`. An easy way to remember it: lvalue reference is infectious, if `&` appears anywhere in the combination, the result is `&`.

Now watch the two cases play out with our `LoudHeavy` class from part 7:
- We pass an **lvalue**: `emplace_back(existing)`. `Args` deduces to `LoudHeavy&`, so `Args&&` is `LoudHeavy& &&`, which collapses to `LoudHeavy&`. The parameter is an lvalue reference.
- We pass an **rvalue**: `emplace_back(LoudHeavy{6})`. `Args` deduces to plain `LoudHeavy`, so `Args&&` is `LoudHeavy&&`. The parameter is an rvalue reference.

One signature, two outcomes, and the deduced `Args` remembers which case we are in. That memory is exactly what `std::forward<Args>` will use later.

## Adding emplace_back() to the RingBuffer
Now that we have touched on the basics of perfect forwarding and variadic templates, we are ready to implement our custom version of `emplace_back()` into our ring buffer.
Here is the declaration inside our `RingBuffer`:
```cpp {hl_lines=[11,12]}
template <typename T, std::size_t N>
class RingBuffer
{
public:
    //Previous code remains the same

    void print_buffer() const;              
    void push_back(const T& new_element);   
    void push_back(T&& new_element);        
    
    template <typename... Args>
    T& emplace_back(Args&&... args);
    
    void pop_front();                       
    bool empty() const;                     
    bool full() const;                      
    
    //Rest of the code remain the same

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
Notice that unlike `push_back`, `emplace_back` returns `T&`, a reference to the freshly built element. This matches what the standard containers do since C++17 and it is convenient: the caller hands us ingredients, we hand back the finished object.

Now for the definition. Inside the function, `args` is a pack of *named* parameters and part 7 taught us exactly what to do with a named parameter we want to move from: cast it back with `std::move`. So our first attempt writes itself:
```cpp {hl_lines=[16]}
template <typename T, std::size_t N>
template <typename... Args>
T& RingBuffer<T, N>::emplace_back(Args&&... args)
{
    if (count_ == N) {
        //Explicitly destroy the oldest element
        slot(0)->~T();
        head_ = (head_ + 1) % N;
    }
    else {
        ++count_;
    }

    //Construct the object directly in the buffer's memory
    //First attempt: a named parameter is an lvalue, so... std::move it?
    T* ptr{new (slot(count_ - 1)) T(std::move(args)...)};

    tail_ = (tail_ + 1) % N;

    //Return a reference to the newly created element
    return *ptr;
}
```
Before we test it, two things about this definition that have nothing to do with forwarding:

First, look at the two `template` headers stacked on top of each other. This is required syntax when we define a member template of a class template outside the class. Two separate templates are in play: the outer header names the class template's parameters (`T`, `N`) and the inner header names the member function's own parameters (`Args`). The class parameters always come first and the two headers cannot be merged into one.

Second, just like our existing `push_back`, this function increments `count_` *BEFORE* constructing the element. If `T`'s constructor throws, `count_` is left one too high and our destructor would later call `~T()` on a slot that never held an object. This is not a new problem that `emplace_back` introduces, it has been lurking in `push_back` since part 6, and we will deal with it properly when we cover the rule of five and exception safety.

## First Test
We reuse `LoudHeavy` from [part 7](ClassesInCpp7) exactly as we left it, the instrumented class whose move constructor visibly *steals* (it takes the pointer and nulls out the source):
```cpp
#include <cstring>
#include <iostream>

struct LoudHeavy {
    int id;
    int* data;
    static constexpr std::size_t data_size{ 1000 };

    LoudHeavy(int i) : id{ i }, data{ new int[data_size] } {
        std::cout << " [Ctro] Created " << id << '\n';
    }

    LoudHeavy(LoudHeavy&& other) : id{ other.id }, data{ other.data } {
        other.data = nullptr;
        other.id = 0;
        std::cout << " [MOVE] Stealing data from " << id << '\n';
    }

    LoudHeavy(const LoudHeavy& other) : id{ other.id }, data{ new int[data_size] } {
        std::memcpy(data, other.data, data_size * sizeof(int));
        std::cout << " [COPY] Duplicating " << id << " (" << data_size * sizeof(int) << " bytes)\n";
    }

    ~LoudHeavy() {
        delete[] data;
        std::cout << " [Dtor] Destroyed " << id << "\n";
    }
};
```
Now the test. We create a `LoudHeavy` that we are NOT done with and pass it to `emplace_back` as an lvalue:
```cpp
#include "ringbuffer.h"

int main()
{
    RingBuffer<LoudHeavy, 4> rb;

    LoudHeavy existing{5};  //A named object we still intend to use
    std::cout << std::boolalpha;
    std::cout << "Before: id = " << existing.id
              << ", data is null: " << (existing.data == nullptr) << '\n';

    rb.emplace_back(existing);  //We pass an LVALUE

    std::cout << "After:  id = " << existing.id
              << ", data is null: " << (existing.data == nullptr) << '\n';
    return 0;
}
```
And the output:
```
 [Ctro] Created 5
Before: id = 5, data is null: false
 [MOVE] Stealing data from 5
After:  id = 0, data is null: true
 [Dtor] Destroyed 0
 [Dtor] Destroyed 5
```
Look at the `After` line. We passed `existing` as an lvalue, we never said we were done with it, and yet our `emplace_back` "gutted it". Its `id` is 0 and its `data` pointer is gone. No compiler error, no crash, no warning. The caller's object was silently stolen, and if `main` had tried to actually use `existing` afterward, it would be reading from a moved-from object.

Why did this happen? `std::move` is an unconditional cast. It does not care that `Args` was deduced as `LoudHeavy&`, it does not care that the caller passed an lvalue, it slams everything to rvalue and the move constructor fires. The part 7 rule, "named parameter, so `std::move` it", was correct for `push_back(T&&)` because there the parameter could ONLY have been bound to an rvalue in the first place. Here, the parameter might be hiding an lvalue, and `std::move` destroys that information.

"Fine", we say, then let's not cast at all. Just pass `args...` plain. That fixes the theft, an lvalue stays an lvalue and gets copied. But now run `rb.emplace_back(LoudHeavy{6})` with a temporary and watch the console: `[COPY] Duplicating 6 (4000 bytes)` where part 7 taught us to expect `[MOVE]`. A named parameter is an lvalue even when it was bound to an rvalue, so the plain version copies everything, and we have thrown away the entire move-semantics.

Let's put the three candidates side by side:

| Argument | `std::move(args)...` | `args...` (plain) | `std::forward<Args>(args)...` |
|---|---|---|---|
| lvalue `existing` | `[MOVE]` — steals it ✗ | `[COPY]` ✓ | `[COPY]` ✓ |
| rvalue `LoudHeavy{6}` | `[MOVE]` ✓ | `[COPY]` — wasteful ✗ | `[MOVE]` ✓ |

`std::move` is right for rvalues and catastrophic for lvalues. Plain `args` is right for lvalues and wasteful for rvalues. Only `std::forward` is correct in both rows, and now the phrase "conditional cast" should land with full weight: it casts to rvalue only when `Args` was deduced from an rvalue, and it preserves lvalue-ness otherwise. The deduced `Args` is the memory, `std::forward<Args>` is the replay.

## std::forward
One line changes:
```cpp {hl_lines=[15]}
template <typename T, std::size_t N>
template <typename... Args>
T& RingBuffer<T, N>::emplace_back(Args&&... args)
{
    if (count_ == N) {
        //Explicitly destroy the oldest element
        slot(0)->~T();
        head_ = (head_ + 1) % N;
    }
    else {
        ++count_;
    }

    //Perfect forwarding: each argument keeps its original value category
    T* ptr{new (slot(count_ - 1)) T(std::forward<Args>(args)...)};

    tail_ = (tail_ + 1) % N;

    //Return a reference to the newly created element
    return *ptr;
}
```
We must always remember that the explicit template argument on `std::forward` is mandatory. It is always `std::forward<Args>(args)`, never bare `std::forward(args)`, the latter will not compile in any useful way. And it is `<Args>`, the deduced pack, not `<T>`. This is the opposite convention from `std::move`, which you call without an explicit type. The reason is that `std::forward` needs to be told what `Args` deduced to, that deduction result is the only record of whether the original argument was an lvalue or an rvalue.

Also note the `...` placement: it sits outside the whole `std::forward<Args>(args)` pattern, so the expansion forwards each argument individually with its own deduced type: `std::forward<A0>(a0), std::forward<A1>(a1), ...`.

Now we rerun the test, this time with one lvalue and one rvalue:
```cpp
#include "ringbuffer.h"

int main()
{
    RingBuffer<LoudHeavy, 4> rb;

    LoudHeavy existing{5};
    rb.emplace_back(existing);          //lvalue -> should COPY
    rb.emplace_back(LoudHeavy{6});      //rvalue -> should MOVE

    std::cout << "existing still valid? id = " << existing.id << '\n';
    return 0;
}
```
And the output:
```
 [Ctro] Created 5
 [COPY] Duplicating 5 (4000 bytes)
 [Ctro] Created 6
 [MOVE] Stealing data from 6
 [Dtor] Destroyed 0
existing still valid? id = 5
 [Dtor] Destroyed 5
 [Dtor] Destroyed 5
 [Dtor] Destroyed 6
```
The lvalue was copied and `existing` survives intact (`id = 5`). The rvalue was moved, and the `[Dtor] Destroyed 0` right after the `[MOVE]` is the gutted temporary `LoudHeavy{6}` dying at the end of its expression, exactly as it should. Both value categories handled correctly by a single function.

By using `Args&&...` and `std::forward<Args>(args)...`, the arguments passed to `T`'s constructor arrive exactly as provided. As a bonus, because `emplace_back` calls the constructor directly rather than relying on an implicit conversion, it also works with `explicit` constructors.

## One Construction Only!
Forwarding correctness was about passing whole objects, but what happens when we pass raw constructor arguments instead? Consider the following:
```cpp
#include "ringbuffer.h"

int main()
{
    RingBuffer<LoudHeavy, 4> rb;

    std::cout << "emplace_back(1):\n";
    rb.emplace_back(1);                 //Builds LoudHeavy in place from the int

    std::cout << "push_back(LoudHeavy{2}):\n";
    rb.push_back(LoudHeavy{2});         //Builds a temporary, then moves it
    return 0;
}
```
And the output:
```
emplace_back(1):
 [Ctro] Created 1
push_back(LoudHeavy{2}):
 [Ctro] Created 2
 [MOVE] Stealing data from 2
 [Dtor] Destroyed 0
 [Dtor] Destroyed 1
 [Dtor] Destroyed 2
```
There it is. `emplace_back(1)` produces exactly one line: `[Ctro] Created 1`. The `int` traveled through the forwarding reference straight into `LoudHeavy`'s constructor, which ran directly inside the buffer's slot. No temporary on the stack, no move, no destructor for a leftover shell. Compare that with the `push_back` block right below it, which is part 7's best case: construct a temporary, move it in, destroy the husk. Three operations versus one. This is the answer to the question we opened with.

## Multiple Arguments, No Default Constructor Needed
Remember `Point` from the bonus section of [part 6](ClassesInCpp6)? Two coordinates required at construction, no default constructor:
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
With `emplace_back`, the parameter pack carries both arguments straight to the constructor:
```cpp
int main()
{
    RingBuffer<Point, 4> points;
    points.emplace_back(3, 4);          //Builds Point(3, 4) in place
    points.emplace_back(5, 6);          //Builds Point(5, 6) in place
    points.print_buffer();
    return 0;
}
```
Output:
```
(3,4) (5,6) 
```
Notice what we did NOT have to write: `points.push_back(Point{3, 4})`. There is no `Point` object anywhere in `main`, only the raw ingredients. The pack deduced `Args` as `int, int` and the expansion delivered both to `Point`'s two-argument constructor inside the slot.

## In the Real World!
Let us create a RingBuffer that we could see in the wild. Remember from [part 1](ClassesInCpp) that logging systems are one of the classic homes of ring buffers: you only care about the most recent entries and memory is bounded.
```cpp
#include "ringbuffer.h"
#include <iostream>
#include <string>

enum class Severity 
{
    Info,
    Warning,
    Error
};

struct LogEntry
{
    long long timestamp;
    Severity level;
    std::string message;

    //Multi-argument constructor
    LogEntry(long long ts, Severity l, std::string msg)
        : timestamp{ts}, level{l}, message{std::move(msg)}
    {
        std::cout << " [Ctor] Created: " << message << '\n';
    }

    LogEntry(const LogEntry& other)
        : timestamp{other.timestamp}, level{other.level}, message{other.message}
    {
        std::cout << " [Copy] Copied!\n";
    }

    LogEntry(LogEntry&& other) noexcept
        : timestamp{other.timestamp}, level{other.level}, message{std::move(other.message)}
    {
        std::cout << " [Move] Moved!\n";
    }

    ~LogEntry() { std::cout << " [Dtor] Destroyed!\n"; }
};

int main()
{
    RingBuffer<LogEntry, 100> logBuffer;

    std::cout << "Calling logBuffer.emplace_back()\n";
    //emplace_back forwards the three raw arguments directly
    //to the LogEntry constructor inside the buffer's slot
    logBuffer.emplace_back(1717852800, Severity::Error, "Connection Lost");

    std::cout << "Calling logBuffer.push_back()\n";
    //push_back requires creating a temporary object first,
    //then moving it into the buffer, then destroying the temporary
    logBuffer.push_back(LogEntry(1717852801, Severity::Info, "Retrying..."));

    return 0;
}
```
And the output:
```
Calling logBuffer.emplace_back()
 [Ctor] Created: Connection Lost
Calling logBuffer.push_back()
 [Ctor] Created: Retrying...
 [Move] Moved!
 [Dtor] Destroyed!
 [Dtor] Destroyed!
 [Dtor] Destroyed!
```
Three constructor arguments of three different types, one of them a `std::string` built from a string literal along the way, and the pack handled all of it. The `emplace_back` path is a single `[Ctor]` line; the `push_back` path pays for the temporary, the move and the husk's destructor, plus the two `[Dtor]` lines at the very end are the buffer cleaning up its two stored entries when `main` returns.

## Does emplace_back Retire push_back?
With a strictly more general tool in hand, you might wonder if `push_back` is now obsolete. It is not, and we will keep both. `push_back(x)` says "store this object I already have", while `emplace_back(parts...)` says "build one from these ingredients". When you genuinely hold a finished object, `push_back` states that intent directly and costs nothing extra. The standard containers kept both for the same reason. With that said, here is the updated RingBuffer:
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
    RingBuffer(const RingBuffer&) = delete;
    RingBuffer(RingBuffer&) = delete;
    RingBuffer& operator=(const RingBuffer&) = delete;
    RingBuffer& operator=(RingBuffer) = delete;


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
    
    template <typename... Args>
    T& emplace_back(Args&&... args);
    
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
    alignas(T) std::byte buffer_[N * sizeof(T)]; //Usually changed to raw bytes for placement new
    std::size_t head_;                         //Index of oldest element (read position)
    std::size_t tail_;                         //Index of next write position
    std::size_t count_;                //Total number of elements in the buffer
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
        //Increment count if we are not overwriting
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

template<typename T, std::size_t N>
template<typename ...Args>
inline T& RingBuffer<T, N>::emplace_back(Args && ...args)
{
    if (count_ == N) {
        //Explicitly destroy the oldest element
        slot(0)->~T();
        head_ = (head_ + 1) % N;
    }
    else {
        ++count_;
    }

    //Construct the object directly in the buffer's memory
    //Perfect forwarding (std::forward) preserves the value category of args
    T* ptr{new (slot(count_ - 1)) T(std::forward<Args>(args)...)};

    tail_ = (tail_ + 1) % N;

    //Return a reference to the newly created element
    return *ptr;
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

## What We Accomplished
We opened with part 7's leftover cost, one temporary and one move per insertion, and eliminated it. Along the way we:
- Learned variadic templates and parameter packs, and demystified pack expansion as pattern repetition.
- Drew the line between a true rvalue reference (`push_back`'s `T&&`, type fixed at class instantiation) and a forwarding reference (`emplace_back`'s `Args&&`, type deduced per call), and saw the reference collapsing rules that make the latter work.
- Walked into the trap part 7 set for us: `std::move(args)...` on a forwarding reference silently steals the caller's lvalues, and watched `LoudHeavy` get gutted in the console to prove it.
- Fixed it with `std::forward<Args>(args)...`, the conditional cast that replays each argument's original value category.
- Implemented `emplace_back` with the double template header, and confirmed the payoff: `emplace_back(1)` is a single construction directly in the buffer's slot.

## What's Next?
Our buffer can now copy, move and build in place, but the `RingBuffer` itself still cannot be copied or moved, we `= delete`d those operations back in [part 6](ClassesInCpp6) as a stopgap. In the next post we will implement them properly and meet the rule of five. That is also where `noexcept` finally gets its moment, you may have noticed `LoudHeavy`'s move constructor still is not marked with it, and where we settle the exception-safety debt we flagged in `emplace_back` and `push_back`. After that: reverse iterators, and the utility methods `back()`, `clear()` and `capacity()`.

# References / Sources
- [Reference declarations (rvalue and forwarding references)](https://en.cppreference.com/w/cpp/language/reference)
- [std::forward](https://en.cppreference.com/w/cpp/utility/forward)
- [Parameter packs](https://en.cppreference.com/w/cpp/language/parameter_pack)
- [new expression (placement new)](https://en.cppreference.com/w/cpp/language/new)
- [Value categories (covered in Part 7)](https://en.cppreference.com/w/cpp/language/value_category)