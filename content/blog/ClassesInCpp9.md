+++
title = "Ring Buffer Series Part 9 — The Rule of Five - deep copies & noexcept"
date = "2026-06-12T12:00:42+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "classes"]

[images]
    featured_image = "/img/blog/ringbuffer9.jpg"
+++

![class](/img/blog/ringbuffer9.jpg)
<br>In [part 6](ClassesInCpp6) we modified the buffer to work by storing bytes, this means that the compiler treats those bytes as raw data and not as the objects that data represents. This means that we also had to do the following:

```cpp
    RingBuffer(const RingBuffer&)            = delete;
    RingBuffer(RingBuffer&&)                 = delete;
    RingBuffer& operator=(const RingBuffer&) = delete;
    RingBuffer& operator=(RingBuffer&&)      = delete;
```
Which effectively makes memberwise copy/move meaningless. Remember, we are using a raw byte buffer and placement new. Because of this we are managing object lifetimes manually. Usually a default compiler-generated copy would simply perform bitwise copy of the `std::byte` array and this creates a few problems. Bitwise copying does not call the actual copy constructors of the `T` objects stored in the buffer, so if `T` is a complex type such as `std::string`, copying its bits without calling the constructor would result in a shallow copy leading to a corrupted state or dangling pointers, if this isn't bad enough, there is also wasted effort as the compiler would copy the entire buffer, including the empty slots that have not been constructed yet. Let us explore some concepts in more detail.
## What is a Shallow Copy?
A shallow copy is an operation that duplicates an object's member variables member by member. When this happens, it means that every data member is copied directly from the source to the destination, if a class contains pointers to dynamic memory, a shallow copy replicates that memory address and not the data being pointed.
We deleted the copy and move constructors as a safety mechanism. Since our class does manual memory management we needed a way to prevent a double destruction as we manually call `~T()` for every object in the buffer. If a bitwise copy is allowed, then we would have two buffers pointing to the same logical resources and then both destructors would eventually run, trying to destroy the same object which in C++ is undefined behavior. 
![ShallowCopy](/img/blog//ShallowCopy.jpg)
Consider the following:
```cpp{{hl_lines=6}}
template <typename T, std::size_t N>
class RingBuffer {
public:
    //Previous code remains the same
    RingBuffer() : head_{}, count_{} {}
    RingBuffer(const RingBuffer&) = default;
    ~RingBuffer() {
        for (std::size_t i = 0; i < count_; ++i) slot(i)->~T();
    }
    template <typename... Args>
    T& emplace_back(Args&&... args) {
        T* p = new (slot(count_)) T(std::forward<Args>(args)...);
        ++count_;
        return *p;
    }
    //Rest of the code remains the same.
private:
    T* slot(std::size_t i) {
        return reinterpret_cast<T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    alignas(T) std::byte buffer_[N * sizeof(T)];
    std::size_t head_;
    std::size_t count_;
};
```
We changed the first constructor from `delete` to `default` in the original buffer, now consider the following test:
```cpp
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

int main() {
    std::cout << "-- fill --\n";
    RingBuffer<LoudHeavy, 4> a;
    a.emplace_back(1);
    a.emplace_back(2);
    std::cout << "-- copy: RingBuffer<LoudHeavy,4> b = a; --\n";
    RingBuffer<LoudHeavy, 4> b = a;
    std::cout << "-- scope ends --\n";
}
```
The code runs, here is the output:
```
-- fill --
 [Ctro] Created 1
 [Ctro] Created 2
-- copy: RingBuffer<LoudHeavy,4> b = a; --
-- scope ends --
 [Dtor] Destroyed 1
 [Dtor] Destroyed 2
 Program returned: 139
 double free or corruption (top)
 Program terminated with signal: SIGSEGV
```
The copy line printed nothing, no `[COPY]`, no `[Ctro]`. The compiler-generated copy just `memcpy`'d the raw byte array, so both buffers now hold the same pointers. When `b` is destroyed it frees those heap blocks, and when `a` is destroyed it tries to free them again, hence the double free.
## What is Deep Copy?
A deep copy is an operation that creates a complete, independent replica of an object and all the resources it manages. While a shallow copy only duplicates members bit by bit, like we already saw, deep copy allocates new resources and duplicates the actual data stored in the source object's addresses. After a deep copy, the original and the new object are distinct from each other, this means that modifications made to the copy do not affect the original object. As we saw, shallow copying is disastrous for an object that manually manages memory, such as our `RingBuffer`. To ensure that our buffer supports deep copy we need to make a series of changes to its constructors.
# The Rule of Five
The rule of five states that if a class requires a custom version of any of the following:
- Destructor
- Copy Constructor
- Copy Assignment Operator
- Move Constructor
- Move Assignment Operator

It pretty much needs to custom define all five and in our case, we have a custom defined destructor, so we must add the following to our RingBuffer.

Before we write any of them, we must ask ourselves: what state actually defines our buffer? Up to now we have been carrying three members: `head_`, `tail_` and `count_`. If we look closely though, `tail_` is never actually read anywhere. Our `slot()` helper computes every position from `head_ + i`, `pop_front` only advances `head_`, and fullness is decided by `count_`. We have been updating `tail_` on every push for no reason at all, it is dead state left over from an earlier design. So the first change we make is to delete it. From here on the buffer has only `head_` and `count_`.

That leaves one decision for the copy: should it reproduce the source's physical layout (the same `head_` offset, elements sitting in the same byte slots) or should it normalize, resetting `head_` to zero and laying the live elements out from the start of the buffer? Both produce a buffer with identical logical contents, but normalizing is cleaner. The physical offset is an implementation detail that nothing outside the class should care about, and replicating it would just be copying bookkeeping for its own sake. So we normalize. A way to think about it is two ring buffers are equal when they hold the same elements in the same order, not when their bytes match.
![CopyNormalized](/img/blog//CopyNormalized.jpg)
### Copy Constructor
```cpp
template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer(const RingBuffer& other) :
    head_{0},
    count_{other.count_}
{
    //Deep-copy only the live elements into the front of our buffer
    for (std::size_t i = 0; i < other.count_; ++i) {
        //Use placement new to call T's copy constructor
        new (slot(i)) T(*other.slot(i));
    }
}
```
The source is walked logically with `other.slot(i)`, which accounts for its `head_` offset, while we write into our own slots `0, 1, 2, ...` starting from a zeroed `head_`. The result is a normalized, independent deep copy.
### Copy Assignment Operator
```cpp
    RingBuffer& operator=(const RingBuffer& other) //Copy Assignment Operator
    {
        //Self-assignment check
        if (this == &other) return *this;

        //Destroy the elements we currently hold
        for (std::size_t i = 0; i < count_; ++i) {
            slot(i)->~T();
        }

        //Start from an empty, normalized buffer and deep-copy other's live elements in
        head_ = 0;
        count_ = 0;
        for (std::size_t i = 0; i < other.count_; ++i) {
            new (slot(i)) T(*other.slot(i));
            ++count_;
        }

        return *this;
    }
```
The code above is not the way a copy assignment operator is usually written. In C++ there is something called the copy-and-swap idiom, which is the standard practice, but unfortunately, this is one of those cases where we need to deviate from standard practices, let's explore why that is.

#### copy-and-swap Idiom
The trick is to write a single assignment operator whose parameter is taken by value:

```cpp
    RingBuffer& operator=(RingBuffer other) //note: 'other' is taken BY VALUE
    {
        std::swap(head_,   other.head_);
        std::swap(count_,  other.count_);
        std::swap(buffer_, other.buffer_); //<-- the problem
        return *this;
        //'other' now holds our OLD state and is destroyed here, cleaning it up
    }
```
Because the parameter is passed by value, the compiler makes the copy for us, using the copy constructor for an lvalue argument, or the move constructor for an rvalue. That means this one operator serves as both copy assignment and move assignment. We then swap our internals with that fresh copy and return; when `other` falls out of scope at the end of the function it carries our old state away and destroys it. It is neat: there is no self-assignment check to remember, and it is naturally safe against exceptions, because all the work that might throw (the copy) happens before we touch `*this`.

So why not use it? Because it leans entirely on `swap` being cheap and valid, and for most classes it is: a swap is just an exchange of a couple of pointers. Our storage is an inline `std::byte` array baked into the object, there is no pointer to hand over. That last line, `std::swap(buffer_, other.buffer_)`, swaps the raw bytes of objects whose constructors never ran on the destination. A correct swap for us would have to move elements one at a time, so copy-and-swap would buy us nothing but extra machinery wrapped around the two operations we are trying to make explicit. We write them out by hand instead.

Not to worry, copy-and-swap will return in the future.
### Move Constructor
```cpp
template<typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>) :
    head_{0},
    count_{0}
{
    //Move each live element into the front of our buffer
    for (std::size_t i = 0; i < other.count_; ++i) {
        //Use placement new and std::move to trigger T's move constructor
        new (slot(i)) T(std::move(*other.slot(i)));
        ++count_;
    }

    //The moved-from husks are still live objects in the source's storage; destroy them
    for (std::size_t i = 0; i < other.count_; ++i) {
        other.slot(i)->~T();
    }

    //Reset the source to a valid, but unspecified empty state
    other.count_ = 0;
    other.head_ = 0;
}
```
This one deserves a second look, because it breaks the intuition we built in [part 7](ClassesInCpp7). There, moving was cheap because a moved-from object usually owns a pointer to its data, and moving just hands that pointer over. Our buffer has no such pointer. The storage is an inline `std::byte` array baked directly into the object, it cannot be detached and handed off. So a `RingBuffer` move cannot be a pointer swap; it has to move each live element individually, exactly like the copy does, only calling `T`'s move constructor instead of its copy constructor.

There is a second subtlety hiding in those husk-destruction loops. Moving from an element does not end its lifetime, the source object is still alive, just in a valid-but-unspecified state (for `LoudHeavy`, that means its `data` pointer is now `nullptr`). Those husks were constructed by us with placement new, so they are still ours to destroy. If we simply set `other.count_ = 0` and walked away, their destructors would never run. For `LoudHeavy` that happens to be harmless because the husk owns nothing, but for any `T` whose moved-from state still holds a resource it would be a leak, and either way it would break the rule we have lived by since [part 6](ClassesInCpp6): every object we placement-new, we destroy exactly once.
### Move Assignment Operator
```cpp
    RingBuffer& operator=(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>) //Move Assignment Operator
    {
        //Self assignment check
        if (this == &other) return *this;

        //Destroy existing objects in this buffer
        for (std::size_t i = 0; i < count_; ++i) {
            slot(i)->~T();
        }

        //Start empty and normalized, then move other's live elements in
        head_ = 0;
        count_ = 0;
        for (std::size_t i = 0; i < other.count_; ++i) {
            //Use placement new and std::move to trigger T's move constructor
            new (slot(i)) T(std::move(*other.slot(i)));
            ++count_;
        }

        //Destroy the moved-from husks before abandoning the source's bookkeeping
        for (std::size_t i = 0; i < other.count_; ++i) {
            other.slot(i)->~T();
        }

        //Reset source to a valid but unspecified empty state
        other.count_ = 0;
        other.head_ = 0;

        return *this;
    }
```
With these changes made we can try the example again and the output we get is this:
```
-- fill --
 [Ctro] Created 1
 [Ctro] Created 2
-- copy: RingBuffer<LoudHeavy,4> b = a; --
 [COPY] Duplicating 1 (4000 bytes)
 [COPY] Duplicating 2 (4000 bytes)
-- scope ends --
 [Dtor] Destroyed 1
 [Dtor] Destroyed 2
 [Dtor] Destroyed 1
 [Dtor] Destroyed 2
```
It works now, no crashes or any kind of errors!
### Moving the whole buffer
The copy works, but the more interesting operation is the move. Let us run the same test with a move instead of a copy:
```cpp
int main() {
    std::cout << "-- fill --\n";
    RingBuffer<LoudHeavy, 4> a;
    a.emplace_back(1);
    a.emplace_back(2);
    std::cout << "-- move: RingBuffer<LoudHeavy,4> b = std::move(a); --\n";
    RingBuffer<LoudHeavy, 4> b = std::move(a);
    std::cout << "a.size() = " << a.size() << ", b.size() = " << b.size() << '\n';
    std::cout << "-- scope ends --\n";
}
```
The output:
```
-- fill --
 [Ctro] Created 1
 [Ctro] Created 2
-- move: RingBuffer<LoudHeavy,4> b = std::move(a); --
 [MOVE] Stealing data from 1
 [MOVE] Stealing data from 2
 [Dtor] Destroyed 0
 [Dtor] Destroyed 0
a.size() = 0, b.size() = 2
-- scope ends --
 [Dtor] Destroyed 1
 [Dtor] Destroyed 2
```
Two `[MOVE]` lines instead of two `[COPY]` lines, exactly as we wanted, no 4000-byte duplications. Then two `[Dtor] Destroyed 0` lines: those are the husks being cleaned up, reporting id `0` because our move constructor zeroed their `id` after stealing the data. Finally the two real elements, now living in `b`, are destroyed when the scope ends, and `a` is reported empty. Every object that was constructed is destroyed exactly once.
## What is noexcept?
The `noexcept` keyword is a specifier that tells the compiler that a function will not throw exceptions. If a function marked as `noexcept` actually throws during runtime, the program is terminated immediately via a call to `std::terminate`.

You may have already spotted `noexcept` on our move constructor and move assignment operator above. That was not decoration, and to understand why we use it we need to learn the `noexcept` operator, and what the standard library does with it.
### The noexcept operator
Confusingly, `noexcept` is two different things wearing the same name. We have already met the specifier, the thing you write on a function to promise it will not throw. There is also the `noexcept` operator, which takes an expression and evaluates, at compile time, to a `bool`: `true` if that expression is known not to throw, `false` otherwise. It does not run the expression, it only inspects its declared exception specification.
```cpp
void safe() noexcept;
void risky();

static_assert(noexcept(safe()) == true);
static_assert(noexcept(risky()) == false);
```
This is the machinery that lets us write a conditional `noexcept` which is a function that is `noexcept` for some types but not for others. That is exactly what we did on our move operations:
```cpp
RingBuffer(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>);
```
`std::is_nothrow_move_constructible_v<T>` is a type trait that asks, at compile time, can a `T` be move-constructed without throwing? If yes, our `RingBuffer`'s move is declared `noexcept`; if no, it is not. The buffer's promise mirrors `T`'s own. This matters because a blanket, unconditional `noexcept` would be a lie for any `T` whose move can throw, and a `noexcept` function that does throw does not propagate the exception, it calls `std::terminate`. Tying our promise to `T`'s guarantee keeps us honest automatically.
### Why move operations care so much
The standard library inspects the `noexcept` status of our move constructor and changes its behavior based on what it finds. The clearest place to see this is a `std::vector` growing.

When a `std::vector` runs out of capacity, it allocates a bigger block and relocates its existing elements into it. To relocate, it would prefer to move each element, that is cheaper. But there is a catch: if a move throws halfway through relocating, the vector is left a wreck, half its elements in the new block, half still in the old one, and a move cannot be cleanly undone. So the library makes a conservative choice. It will move elements only if their move constructor is `noexcept`; otherwise it falls back to copying them, because a copy that throws leaves the original untouched and the vector can recover. This decision is made through `std::move_if_noexcept`.

Let us watch it happen. With `LoudHeavy`'s move constructor left un-marked:
```cpp
int main() {
    std::vector<LoudHeavy> v;
    v.reserve(2);
    std::cout << "-- emplace 1 --\n"; v.emplace_back(1);
    std::cout << "-- emplace 2 --\n"; v.emplace_back(2);
    std::cout << "-- emplace 3 (forces reallocation) --\n"; v.emplace_back(3);
    std::cout << "-- main ends --\n";
}
```
We reserve room for two, then push a third, forcing the vector to grow. The output:
```
-- emplace 1 --
 [Ctro] Created 1
-- emplace 2 --
 [Ctro] Created 2
-- emplace 3 (forces reallocation) --
 [Ctro] Created 3
 [COPY] Duplicating 1 (4000 bytes)
 [COPY] Duplicating 2 (4000 bytes)
 [Dtor] Destroyed 1
 [Dtor] Destroyed 2
-- main ends --
 [Dtor] Destroyed 1
 [Dtor] Destroyed 2
 [Dtor] Destroyed 3
```
Look at what relocation cost us: two `[COPY]` lines, 4000 bytes duplicated each, even though `LoudHeavy` has a perfectly good move constructor sitting right there. The vector refused to use it, because it was not promised to be safe.

Now we add a single word to `LoudHeavy`'s move constructor:
```cpp
LoudHeavy(LoudHeavy&& other) noexcept : id{ other.id }, data{ other.data } {
    other.data = nullptr;
    other.id = 0;
    std::cout << " [MOVE] Stealing data from " << id << '\n';
}
```
Same program, nothing else changed. The output:
```
-- emplace 1 --
 [Ctro] Created 1
-- emplace 2 --
 [Ctro] Created 2
-- emplace 3 (forces reallocation) --
 [Ctro] Created 3
 [MOVE] Stealing data from 1
 [Dtor] Destroyed 0
 [MOVE] Stealing data from 2
 [Dtor] Destroyed 0
-- main ends --
 [Dtor] Destroyed 1
 [Dtor] Destroyed 2
 [Dtor] Destroyed 3
```
The copies are gone. The vector now moves, stealing each pointer instead of duplicating 4000 bytes, and immediately destroys each husk (the `Destroyed 0` lines). One keyword turned a deep copy into a handful of pointer swaps. That is the whole reason `noexcept` belongs on a move constructor: not because it makes that function faster, but because it unlocks the move path in every container that holds your type.

### Marking the rest of our members
With the move operations handled, we should mark the rest of the class. The accessors that obviously cannot throw should say so:
```cpp
std::size_t size() const noexcept;
bool empty() const noexcept;
bool full() const noexcept;
void pop_front() noexcept;
```
`pop_front` only destroys an element and advances an index, neither of which throws. `size`, `empty` and `full` are trivial reads. The copy constructor and copy assignment operator are deliberately not `noexcept`, copying a `LoudHeavy` allocates, and allocation can throw. 

We also leave `front()` unmarked. Today it just returns a reference and could safely be `noexcept`, but it is the one function we might later want to throw from, a bounds-checked version that rejected an empty buffer with an exception, and leaving that door open now is cheaper than reopening it later.

Here is the updated buffer's code:
```cpp
#ifndef RINGBUFFER_H
#define RINGBUFFER_H
#include <cstddef>      //std::size_t, std::byte, std::ptrdiff_t
#include <iterator>     //std::forward_iterator_tag
#include <new>          //placement new
#include <utility>      //std::move, std::forward
#include <cassert>      //assert
#include <type_traits>  //std::is_nothrow_move_constructible_v
#include <iostream>     //std::cout

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

    //Const overload for read-only contexts like `print_buffer`
    const_iterator begin() const { return const_iterator(this, 0); }
    const_iterator end() const { return const_iterator(this, count_); }

    //Explicitly constant iterators
    const_iterator cbegin() const { return begin(); }
    const_iterator cend() const { return end(); }

    RingBuffer();       //Constructor
    RingBuffer(const RingBuffer& other); //Copy Constructor
    RingBuffer(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>); //Move Constructor

    RingBuffer& operator=(const RingBuffer& other) //Copy Assignment Operator
    {
        //Self-assignment check
        if (this == &other) return *this;

        //Destroy the elements we currently hold
        for (std::size_t i = 0; i < count_; ++i) {
            slot(i)->~T();
        }

        //Start from an empty, normalized buffer and deep-copy other's live elements in
        head_ = 0;
        count_ = 0;
        for (std::size_t i = 0; i < other.count_; ++i) {
            new (slot(i)) T(*other.slot(i));
            ++count_;
        }

        return *this;
    }

    RingBuffer& operator=(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>) //Move Assignment Operator
    {
        //Self assignment check
        if (this == &other) return *this;

        //Destroy existing objects in this buffer
        for (std::size_t i = 0; i < count_; ++i) {
            slot(i)->~T();
        }

        //Start empty and normalized, then move other's live elements in
        head_ = 0;
        count_ = 0;
        for (std::size_t i = 0; i < other.count_; ++i) {
            //Use placement new and std::move to trigger T's move constructor
            new (slot(i)) T(std::move(*other.slot(i)));
            ++count_;
        }

        //Destroy the moved-from husks before abandoning the source's bookkeeping
        for (std::size_t i = 0; i < other.count_; ++i) {
            other.slot(i)->~T();
        }

        //Reset source to a valid but unspecified empty state
        other.count_ = 0;
        other.head_ = 0;

        return *this;
    }
    ~RingBuffer();      //Destructor

    void print_buffer() const;              //Outputs the buffers contents to the screen
    void push_back(const T& new_element);   //Adds a new element into the buffer
    void push_back(T&& new_element);        //Adds new element using an rvalue reference

    template <typename... Args>
    T& emplace_back(Args&&... args);

    void pop_front() noexcept;              //Deletes the oldest element from the buffer
    bool empty() const noexcept;            //Is the buffer empty
    bool full() const noexcept;             //Is the buffer full
    const T& front() const;                 //Retrieves the oldest element
    std::size_t size() const noexcept;      //Retrieves the number of elements in the buffer

private:
    T* slot(std::size_t i) {
        return reinterpret_cast<T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    const T* slot(std::size_t i) const {
        return reinterpret_cast<const T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    alignas(T) std::byte buffer_[N * sizeof(T)]{}; //Raw, aligned storage for placement new
    std::size_t head_;                         //Index of oldest element (read position)
    std::size_t count_;                        //Total number of elements in the buffer
};

template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer() : head_{}, count_{} {}

template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer(const RingBuffer& other) :
    head_{ 0 },
    count_{ other.count_ }
{
    //Deep-copy only the live elements into the front of our buffer
    for (std::size_t i = 0; i < other.count_; ++i) {
        //Use placement new to call T's copy constructor
        new (slot(i)) T(*other.slot(i));
    }
}

template<typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>) :
    head_{ 0 },
    count_{ 0 }
{
    //Move each live element into the front of our buffer
    for (std::size_t i = 0; i < other.count_; ++i) {
        //Use placement new and std::move to trigger T's move constructor
        new (slot(i)) T(std::move(*other.slot(i)));
        ++count_;
    }

    //The moved-from husks are still live objects in the source's storage; destroy them
    for (std::size_t i = 0; i < other.count_; ++i) {
        other.slot(i)->~T();
    }

    //Reset the source to a valid, but unspecified empty state
    other.count_ = 0;
    other.head_ = 0;
}

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
}

template<typename T, std::size_t N>
template<typename ...Args>
T& RingBuffer<T, N>::emplace_back(Args && ...args)
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
    T* ptr{ new (slot(count_ - 1)) T(std::forward<Args>(args)...) };

    //Return a reference to the newly created element
    return *ptr;
}

template <typename T, std::size_t N>
void RingBuffer<T, N>::pop_front() noexcept
{
    if (count_ == 0) return;

    //Manually end the lifetime of the oldest element
    slot(0)->~T();

    head_ = (head_ + 1) % N;
    --count_;
}

template <typename T, std::size_t N>
bool RingBuffer<T, N>::empty() const noexcept
{
    return count_ == 0;
}

template <typename T, std::size_t N>
bool RingBuffer<T, N>::full() const noexcept
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
std::size_t RingBuffer<T, N>::size() const noexcept
{
    return count_;
}
#endif // RINGBUFFER_H
```
## Wrapping Up
Our `RingBuffer` is now a fully-fledged value type. It can be copied, moved, copy-assigned and move-assigned; every one of those operations does the right thing with our manual byte storage; and our move operations are `noexcept` whenever the element type's are, which means a `RingBuffer` slots cleanly into containers like `std::vector` and gets relocated by move rather than copy. We satisfied the rule of five, and along the way we threw out a member we never needed.

There is, however, a crack we have papered over. Look again at the loop inside our copy constructor: we placement-new each element one at a time. What happens if the third of five copies throws? The first two are already constructed, but the object's constructor has not finished, so its destructor will never run, and those two elements leak. The same fragility lives in `push_back` and `emplace_back`, where we increment `count_` before we construct the element, so a throwing constructor leaves the buffer believing it holds an object that was never built, and our destructor will later try to destroy it.

That is the subject of the next post: exception safety. We will give names to the guarantees a function can offer (basic, strong, nothrow), build a small instrument whose constructor throws on demand, watch our current code corrupt itself, and then fix it with the classic construct-first, commit-later reordering.
## References
- [Rule of three/five/zero](https://en.cppreference.com/w/cpp/language/rule_of_three) — cppreference
- [Copy constructors](https://en.cppreference.com/w/cpp/language/copy_constructor) — cppreference
- [Copy assignment operator](https://en.cppreference.com/w/cpp/language/copy_assignment) — cppreference
- [Move constructors](https://en.cppreference.com/w/cpp/language/move_constructor) — cppreference
- [Move assignment operator](https://en.cppreference.com/w/cpp/language/move_assignment) — cppreference
- [`noexcept` specifier](https://en.cppreference.com/w/cpp/language/noexcept_spec) — cppreference
- [`noexcept` operator](https://en.cppreference.com/w/cpp/language/noexcept) — cppreference
- [`std::is_nothrow_move_constructible`](https://en.cppreference.com/w/cpp/types/is_move_constructible) — cppreference
- [`std::move_if_noexcept`](https://en.cppreference.com/w/cpp/utility/move_if_noexcept) — cppreference
- [`std::terminate`](https://en.cppreference.com/w/cpp/error/terminate) — cppreference