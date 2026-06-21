+++
title = "Ring Buffer Series Part 10 — Exception Safety: naming the guarantees & fixing our bookkeeping"
date = "2026-06-16T12:00:42+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "classes"]

[images]
    featured_image = "/img/blog/ringbuffer10.jpg"
+++

![class](/img/blog/ringbuffer10.jpg)
<br>At the end of [part 9](ClassesInCpp9) we realized that our buffer still had one issue. Our copy constructor placement-news each element one at a time; if the third of five copies throws, the first two are already built, but because the *constructor itself* failed, the object's destructor will never run, and those two elements leak. The same fragility lives in `push_back` and `emplace_back`: we increment `count_` *before* we construct the element, so a throwing constructor leaves the buffer believing it holds an object that was never built, and our destructor will later walk into that slot and try to destroy it.
![OneThrow](/img/blog/OneThrow.jpg)

Today, we are here to correct that issue and we will do so based on a simple idea: keep `count_` equal to the number of elements that are actually alive, and commit that change only after the step that can throw has succeeded. Before we can talk about the fixes, though, we need to dive deeper into everything that makes "exception safety" possible, let's start with the basics:
## What is the Call Stack?
Sometimes just called the Stack, it is a region of memory reserved by the operating system to support function execution. We can think of it as a to-do list of functions of our program. It keeps track of which functions are currently running and where to resume once they finish. A common analogy is stack of plates at a restaurant. When we call a function, we push a new plate on top of the pile. This plate holds everything that the function needs: its local variables, parameters and a bookmark called the return address, so the program knows where to go back to. The program only ever works on the plate at the very top. When that function finishes, its plate is popped off and its memory instantly and automatically reclaimed. The CPU then uses the bookmark to resume the function on the plate immediately underneath.
![CallStack](/img/blog/CallStack.jpg)

## What is an Exception?
An exception is a tool that we use to notify one part of our code that an "exceptional" situation or error happened in another part. There are usually three parts to how exceptions work. We start with what we call a throw, this is when a function detects a problem it cannot solve, so it "throws" an object representing the error. At this point, the normal step-by-step execution stops and the program begins searching for a handler. As it searches it unwinds the call stack, walking back up through the active function calls and destroying their local objects along the way. When it finds a matching handler (a `catch` whose type fits the thrown object), control jumps there and the handler runs. If it reaches the top of the stack without finding one, it calls `std::terminate` and end the program, for users of our software, this is usually known as a crash.

## Exception Safety Guarantees
This is a formal contract regarding the state of our program and its objects after an exception is thrown. It tells a function how to behave when it can't complete its assigned task, making sure that the system stays predictable and not falling into an undefined or corrupted state. We have the following levels of exception safety:

- **No-throw (nothrow):** the function will not throw at all. If it cannot do its job, it reports failure some other way (or simply cannot fail). This is the promise `noexcept` makes machine-checkable. In our buffer, `pop_front`, `size`, `empty`, and `full` already make this promise, and our move operations make it whenever the element type's move does.
- **Strong guarantee:** commit-or-rollback. If the function throws, the object is left exactly as it was before the call, as though the operation never happened. Nothing is lost, nothing is half-done.
- **Basic guarantee:** if the function throws, nothing leaks and the object is left in a valid state, though possibly a different one than it started in, and not necessarily one you can predict. Invariants hold; the object can still be used and safely destroyed.
- **No guarantee:** a throw can leave the object corrupted, leaking memory, holding broken invariants, or set up for undefined behavior later. This is where our current code lives, as we are about to see.

A useful way to think about it is: no-throw says "I won't fail," strong says "if I fail, it's as if I never tried," basic says "if I fail, I'll at least leave you something usable," and no guarantee says "if I fail, you're on your own."

You may be wondering what `noexcept` has to do with all of this since we spent a long time on it in part 9. `noexcept` is the no-throw guarantee, expressed in a way the compiler enforces. A function can offer the strong or basic guarantee while still being perfectly entitled to throw. `noexcept` is specifically the promise "and I will never throw," with `std::terminate` as the penalty for lying.

## Introducing a throwing object
To watch our code corrupt itself we need a type that throws exactly when we tell it to, and that lets us see the damage. `LoudHeavy` from parts 7 through 9 prints its special operations, but it cannot fail on cue, and its heap pointer means a botched teardown crashes instead of reporting. So we build a second instrument, `Throwing`:

```cpp
#include <stdexcept>
#include <iostream>

struct Throwing {
    int id;
    static inline int live        = 0;   // objects currently alive - the leak detector
    static inline int constructed = 0;   // total ctor/copy attempts
    static inline int throw_at    = -1;  // which attempt throws (-1 = never)
    static void arm(int nth) { live = 0; constructed = 0; throw_at = nth; }

    explicit Throwing(int i) : id{ i } { check(); ++live; std::cout << " [Ctro] " << id << "  (live=" << live << ")\n"; }
    Throwing(const Throwing& o) : id{ o.id } { check(); ++live; std::cout << " [COPY] " << id << "  (live=" << live << ")\n"; }
    Throwing(Throwing&& o) noexcept : id{ o.id } { ++live; std::cout << " [MOVE] " << id << "  (live=" << live << ")\n"; }
    ~Throwing() { --live; std::cout << " [Dtor] " << id << "  (live=" << live << ")\n"; }
private:
    static void check() {
        if (++constructed == throw_at) {
            std::cout << " [THROW] on construction attempt #" << constructed << '\n';
            throw std::runtime_error("Throwing failed");
        }
    }
};
```

The three `static inline` members are the whole point. `live` is incremented in every constructor and decremented in the destructor, so at any moment it equals the number of `Throwing` objects in existence. After any correct sequence of operations it returns to where it started. If we leak an object, `live` is left too high. If we destroy a slot that was never constructed, `live` is driven negative, and is therefore a signal that something destroyed memory it should not have touched. `constructed` counts every construction attempt (the ordinary constructor and the copy constructor both call `check`), and `throw_at` is the attempt number on which `check` throws.

`arm()` resets `live` to zero, so we must arm the instrument before we create any `Throwing` objects. Every demo below arms first, then builds. Unlike `LoudHeavy`, `Throwing` owns no heap memory. That is what lets us safely observe a botched teardown: when our buggy destructor walks into a slot that was never constructed, `~Throwing` merely decrements `live` and prints, rather than calling `delete[]` on a garbage pointer and taking the process down with it. The bug is still undefined behavior, but it is observable undefined behavior rather than an immediate crash.

Each demo catches the exception in `main`. If we let it escape, an uncaught exception calls `std::terminate`, the program dies, and we lose both the `live` counter and the teardown we are trying to watch.

## The Phantom Slot
Recall our `emplace_back` from part 9:
```cpp
template<typename T, std::size_t N>
template<typename ...Args>
T& RingBuffer<T, N>::emplace_back(Args && ...args)
{
    if (count_ == N) {
        slot(0)->~T();
        head_ = (head_ + 1) % N;
    }
    else {
        ++count_;                                       // <-- count_ committed BEFORE the object exists
    }
    T* ptr{ new (slot(count_ - 1)) T(std::forward<Args>(args)...) };  // <-- if this throws, count_ is already too high
    return *ptr;
}
```

Look at the non-full path. We increment `count_`, then construct into `slot(count_ - 1)`. If that construction throws, control leaves the function with `count_` already bumped but no object in the slot. The buffer now claims to hold an element that was never built. Let's arm `Throwing` to throw on the second construction and watch:

```cpp
int main() {
    Throwing::arm(2);                      // the 2nd construction will throw
    RingBuffer<Throwing, 8> buf;
    buf.emplace_back(1);                   // construction #1 - fine
    try {
        buf.emplace_back(2);               // construction #2 - throws
    } catch (const std::exception& e) {
        std::cout << " caught: " << e.what() << '\n';
    }
    std::cout << " size() reports: " << buf.size() << '\n';
    std::cout << " live objects:   " << Throwing::live << '\n';
    std::cout << "-- buf goes out of scope --\n";
}
```

The output:

```
 [Ctro] 1  (live=1)
 [THROW] on construction attempt #2
 caught: Throwing failed
 size() reports: 2
 live objects:   1
-- buf goes out of scope --
 [Dtor] 1  (live=0)
 [Dtor] 2  (live=-1)
```

The discrepancy is right there in the middle: `size()` reports 2, but only 1 object is actually alive. The `else { ++count_; }` ran, the construction did not. We caught the exception, the program kept going, and the buffer is now lying about its own length.

The lie turns into undefined behavior at the bottom. When `buf` is destroyed, our destructor loops `for (i = 0; i < count_; ++i)` and calls `~T()` on each slot. With `count_` stuck at 2 it destroys the real element (`id 1`), then reaches into the second slot, which was never constructed, and calls `~Throwing` on it anyway. `live` goes to -1. This means we have run one more destructor than constructor.

Notice something very important. The phantom destructor prints `[Dtor] 2`, not `[Dtor] 0`. The buffer's bytes are zero-initialized, so you might expect the never-constructed slot to read as `id 0`. But `Throwing(int)` sets `id{ i }` in its member-initializer list, which runs before the body's `check()` throws. So the failed construction still wrote `2` into those bytes before bailing out. The slot's lifetime never legally began, but the value is there, and our destructor dutifully reads it. The headline is not the id, it is the `live = -1`. This operation currently offers no guarantee at all.

## Construct First, Commit Later
The non-full path is the easy and satisfying case. The order of two statements is wrong, and swapping them buys us the strong guarantee for free, with no `try`/`catch` and no cleanup code:

```cpp
new (slot(count_)) T(std::forward<Args>(args)...);   // the one step that can throw
++count_;                                             // commit only after it succeeds
```

If the construction throws now, `count_` was never touched. The stack unwinds, the function exits, and the buffer is byte-for-byte what it was before the call. The element we tried to add simply is not there, exactly as if we had never called `emplace_back`. That is the strong guarantee, and we got it by reordering, not by writing recovery logic. The same `Throwing::arm(2)` demo against the fixed code:

```
 [Ctro] 1  (live=1)
 [THROW] on construction attempt #2
 caught: Throwing failed
 size() reports: 1
 live objects:   1
-- buf goes out of scope --
 [Dtor] 1  (live=0)
```

`size()` reports 1, `live` is 1, they agree, and teardown is clean. No phantom.

## What Happens if the Buffer is Full?
When the buffer is full, every one of the `N` slots is occupied. To make room for the new element we must first destroy the oldest one:

```cpp
if (count_ == N) {
    slot(0)->~T();              // destroy the oldest element
    head_ = (head_ + 1) % N;    // advance past it
}
new (slot(count_ - 1)) T(...);  // construct the replacement into the slot we just freed
```

Read that sequence carefully. We destroy the old element before we construct the new one, because there is no spare slot to build the new one in first. And `~T()` is irreversible: once the oldest element's destructor has run, that element is gone. There is no copy of it anywhere. So if the construction on the next line throws, we have already lost the old element and there is nothing to roll back to. The strong guarantee, "as if the call never happened," is unreachable here without somewhere to stash a spare.

![FullPathBasic](/img/blog/FullPathBasic.jpg)
The bug today is not that we lose an element, that is unavoidable, it is that a throw leaves `count_` wrong and our destructor reaching into garbage. The fix is to commit the eviction the moment it happens, by decrementing `count_` as soon as the old element is destroyed. Then the buffer is a valid, smaller buffer, and the construction that follows is just the ordinary non-full case. Both paths collapse into the same construct-then-commit tail:

```cpp
template <typename T, std::size_t N>
void RingBuffer<T, N>::push_back(const T& new_element)
{
    if (count_ == N) {                       // overwrite: commit the eviction FIRST
        slot(0)->~T();
        head_ = (head_ + 1) % N;
        --count_;                            // the oldest is gone; record it now
    }
    new (slot(count_)) T(new_element);        // the one step that can throw
    ++count_;                                 // commit only on success
}
```

When the buffer is not full, the `if` is skipped and we get the strong guarantee from the previous section. When it is full, we destroy the oldest, drop `count_` to `N - 1`, leaving a perfectly valid buffer of `N - 1` elements, and then construct into the now-free slot. If that throws, we are left with that valid `N - 1` buffer: we lost the oldest element, the new one never arrived, but nothing leaked and nothing is corrupt. Consider the following:

```cpp
int main() {
    Throwing::arm(3);
    RingBuffer<Throwing, 2> buf;
    buf.emplace_back(1);                   // #1
    buf.emplace_back(2);                   // #2 - buffer is now full
    try {
        buf.emplace_back(3);               // evicts id 1, then construction of id 3 throws
    } catch (const std::exception& e) {
        std::cout << " caught: " << e.what() << '\n';
    }
    std::cout << " size() reports: " << buf.size() << '\n';
    std::cout << " live objects:   " << Throwing::live << '\n';
    std::cout << " front() is now: " << buf.front().id << '\n';
    std::cout << "-- buf goes out of scope --\n";
}
```

```
 [Ctro] 1  (live=1)
 [Ctro] 2  (live=2)
 [Dtor] 1  (live=1)
 [THROW] on construction attempt #3
 caught: Throwing failed
 size() reports: 1
 live objects:   1
 front() is now: 2
-- buf goes out of scope --
 [Dtor] 2  (live=0)
```

`id 1` was evicted (`[Dtor] 1` before the throw), the replacement `id 3` never made it, and the buffer is left holding exactly `id 2`: `size()` and `live` both report 1, `front()` returns 2, and teardown is clean. `live` never goes negative. This is the basic guarantee made concrete: we lost data we cannot recover, but the object remains valid and leak-free. The non-full path is strong; the full path is basic; and now we know precisely why the difference exists, it is the missing spare slot. (Recovering the strong guarantee on the full path is possible, but only by building the replacement somewhere else first, which means either extra storage or leaning on a `noexcept` move.)

## The Leaking Copy

The second bug from part 9 is in the copy constructor. Here it is again:
```cpp
template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer(const RingBuffer& other) :
    head_{ 0 },
    count_{ other.count_ }          // <-- we claim the full count up front
{
    for (std::size_t i = 0; i < other.count_; ++i) {
        new (slot(i)) T(*other.slot(i));   // <-- if copy k throws, copies 0..k-1 are already built
    }
}
```

We set `count_` to the source's count immediately, then copy element by element. Arm `Throwing` so the buffer has two elements and the second copy fails:

```cpp
int main() {
    Throwing::arm(4);                      // the 4th construction will throw
    RingBuffer<Throwing, 8> src;
    src.emplace_back(1);                   // #1
    src.emplace_back(2);                   // #2
    try {
        RingBuffer<Throwing, 8> dst(src);  // copies id 1 (#3), then id 2 (#4) throws
    } catch (const std::exception& e) {
        std::cout << " caught: " << e.what() << '\n';
    }
    std::cout << " live objects (src holds 2): " << Throwing::live << '\n';
    std::cout << "-- src goes out of scope --\n";
}
```

```
 [Ctro] 1  (live=1)
 [Ctro] 2  (live=2)
 [COPY] 1  (live=3)
 [THROW] on construction attempt #4
 caught: Throwing failed
 live objects (src holds 2): 3
-- src goes out of scope --
 [Dtor] 1  (live=2)
 [Dtor] 2  (live=1)
```

Here we have the opposite problem of the phantom slot. There, `live` went negative, here it stays too high. `dst` successfully copied `id 1` (`[COPY] 1`, `live` rises to 3), then the copy of `id 2` threw. At that point `dst` held one fully-constructed element. But because `dst`'s constructor threw, `dst` is not a constructed object, so its destructor never runs, and that one copied element is orphaned. `live` reads 3 when only `src`'s two objects should exist. When `src` is destroyed at the end of `main`, `live` walks down to 1 and stays there: the leaked copy is never reclaimed and we are all left very sad.

## Why a Constructor Can't Lean on its Destructor
Our copy assignment operator uses essentially the same element-by-element loop, incrementing `count_` after each copy, and yet it does not leak. Why does the same pattern that is safe in assignment fail in construction? To answer that question we must understand whose destructor we are actually running.

When `operator=` is executing, `*this` is a fully-constructed, live object. It was constructed at some earlier point and it will be destroyed at some later point no matter what happens inside the assignment. So if a copy partway through the rebuild throws, the function exits, but `*this` is still a live object with `count_` reflecting however many elements it managed to rebuild, and its destructor will eventually run and clean up exactly those elements. The per-element `++count_` is what makes that work: it keeps `count_` accurate at every step, so whenever the destructor fires it destroys precisely the elements that exist. Assignment can lean on its own destructor as a safety net.

A constructor has no such net. While a constructor is running, the object does not yet exist; its lifetime only begins if the constructor completes. If the constructor throws, the language guarantees the object's destructor is never called, precisely because, from the language's point of view, there was never a finished object to destroy. So incrementing `count_` inside a throwing constructor accomplishes nothing, there is no destructor coming to read it. Whatever the constructor built, it must tear down itself, by hand, before it lets the exception propagate.

This same asymmetry explains why the `push_back` fix needed no `try`/`catch` but this one will. `push_back` has a single fallible step, so reordering is enough: do the throwing thing, and only commit if it returns. The copy constructor builds a batch of elements, any one of which can throw, and on failure it has a partially-built batch to dispose of.

## Cleaning Up by Hand

Since the constructor must do its own cleanup, we wrap the loop in a `try`/`catch` that destroys whatever we managed to build and then rethrows. The trick that keeps it simple is to drive the loop with `count_` itself, so that at the moment of a throw, `count_` already holds the number of live elements, and the cleanup loop knows exactly how far to go:

```cpp
template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer(const RingBuffer& other) :
    head_{ 0 },
    count_{ 0 }                                          // start empty; grow as we build
{
    try {
        for (; count_ < other.count_; ++count_) {
            new (slot(count_)) T(*other.slot(count_));   // count_ tracks the live elements
        }
    } catch (...) {
        for (std::size_t j = 0; j < count_; ++j) {       // unwind the partial batch
            slot(j)->~T();
        }
        throw;                                           // rethrow: the object never lived
    }
}
```

`count_` starts at zero and is incremented only after a successful copy, so it is always equal to the number of constructed elements. If a copy throws, we drop into the `catch`, destroy slots `0` through `count_ - 1`, and rethrow the original exception unchanged (`throw;` with no operand). Notice this is the exact same invariant as the `push_back` fix, `count_` equals the live count at all times, just enforced manually because no destructor is coming to help. With the fixed constructor, we run the demo again:

```
 [Ctro] 1  (live=1)
 [Ctro] 2  (live=2)
 [COPY] 1  (live=3)
 [THROW] on construction attempt #4
 [Dtor] 1  (live=2)
 caught: Throwing failed
 live objects (src holds 2): 2
-- src goes out of scope --
 [Dtor] 1  (live=1)
 [Dtor] 2  (live=0)
```

The `[Dtor] 1` immediately after the throw, before `caught` prints, is our `catch` block destroying the one copy that succeeded. `live` returns to 2 (just `src`), and after `src` is destroyed it reaches 0. No leak. The copy constructor now offers the basic guarantee: if it fails, it cleans up after itself and propagates the error, leaving no trace.

## Our Buffer's Guarantees

| Operation | Guarantee | Why |
| --- | --- | --- |
| `size`, `empty`, `full`, `pop_front` | no-throw | marked `noexcept`; touch only `count_`/`head_` |
| move ctor / move-assign | no-throw *when `T`'s move is* | conditional `noexcept` from part 9 |
| `push_back` / `emplace_back`, not full | strong | construct first, commit `count_` after |
| `push_back` / `emplace_back`, overwriting | basic | the evicted element cannot be recovered |
| copy constructor | basic | `try`/`catch` cleans up the partial batch |
| copy-assignment | basic | destroys current contents, then rebuilds |

However, we are still not happy. Our copy-assignment is only basic and it is destructive on failure: it destroys the buffer's current contents before it starts rebuilding, so a throw partway through the rebuild loses both the old contents and the new. To make assignment strong, we would need to build a complete copy first, off to the side, where a failure cannot touch `*this`, and only then commit. 

Here is the updated Buffer's code:

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
        using iterator_category = std::forward_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = T*;
        using reference = T&;
        T& operator*() const { return *rb_->slot(index_); }
        pointer operator->() const { return rb_->slot(index_); }
        iterator& operator++() { ++index_; return *this; }
        bool operator!=(const iterator& other) const { return index_ != other.index_; }
        bool operator==(const iterator& other) const { return rb_ == other.rb_ && index_ == other.index_; }
    private:
        friend class RingBuffer;
        iterator(RingBuffer* rb, std::size_t index) : rb_{ rb }, index_{ index } {}
        RingBuffer* rb_;
        std::size_t index_;
    };

    class const_iterator {
    public:
        using iterator_category = std::forward_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = const T*;
        using reference = const T&;
        const T& operator*() const { return *rb_->slot(index_); }
        pointer operator->() const { return rb_->slot(index_); }
        const_iterator& operator++() { ++index_; return *this; }
        bool operator!=(const const_iterator& other) const { return index_ != other.index_; }
        bool operator==(const const_iterator& other) const { return rb_ == other.rb_ && index_ == other.index_; }
    private:
        friend class RingBuffer;
        const_iterator(const RingBuffer* rb, std::size_t index) : rb_{ rb }, index_{ index } {}
        const RingBuffer* rb_;
        std::size_t index_;
    };

    iterator begin() { return iterator(this, 0); }
    iterator end() { return iterator(this, count_); }
    const_iterator begin() const { return const_iterator(this, 0); }
    const_iterator end() const { return const_iterator(this, count_); }
    const_iterator cbegin() const { return begin(); }
    const_iterator cend() const { return end(); }

    RingBuffer();
    RingBuffer(const RingBuffer& other);
    RingBuffer(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>);

    RingBuffer& operator=(const RingBuffer& other)   //Copy Assignment - basic guarantee
    {
        if (this == &other) return *this;
        for (std::size_t i = 0; i < count_; ++i) {
            slot(i)->~T();
        }
        head_ = 0;
        count_ = 0;
        for (std::size_t i = 0; i < other.count_; ++i) {
            new (slot(i)) T(*other.slot(i));
            ++count_;                                 // *this is alive; its dtor cleans up on a throw
        }
        return *this;
    }

    RingBuffer& operator=(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>)  //Move Assignment
    {
        if (this == &other) return *this;
        for (std::size_t i = 0; i < count_; ++i) {
            slot(i)->~T();
        }
        head_ = 0;
        count_ = 0;
        for (std::size_t i = 0; i < other.count_; ++i) {
            new (slot(i)) T(std::move(*other.slot(i)));
            ++count_;
        }
        for (std::size_t i = 0; i < other.count_; ++i) {
            other.slot(i)->~T();
        }
        other.count_ = 0;
        other.head_ = 0;
        return *this;
    }
    ~RingBuffer();

    void print_buffer() const;
    void push_back(const T& new_element);
    void push_back(T&& new_element);

    template <typename... Args>
    T& emplace_back(Args&&... args);

    void pop_front() noexcept;
    bool empty() const noexcept;
    bool full() const noexcept;
    const T& front() const;
    std::size_t size() const noexcept;

private:
    T* slot(std::size_t i) {
        return reinterpret_cast<T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    const T* slot(std::size_t i) const {
        return reinterpret_cast<const T*>(&buffer_[((head_ + i) % N) * sizeof(T)]);
    }
    alignas(T) std::byte buffer_[N * sizeof(T)]{};
    std::size_t head_;
    std::size_t count_;
};

template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer() : head_{}, count_{} {}

template <typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer(const RingBuffer& other) :
    head_{ 0 },
    count_{ 0 }
{
    //Build up the copy element by element; count_ always equals the number alive
    try {
        for (; count_ < other.count_; ++count_) {
            new (slot(count_)) T(*other.slot(count_));
        }
    }
    catch (...) {
        //A throwing constructor's destructor never runs, so clean up by hand
        for (std::size_t j = 0; j < count_; ++j) {
            slot(j)->~T();
        }
        throw;
    }
}

template<typename T, std::size_t N>
RingBuffer<T, N>::RingBuffer(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>) :
    head_{ 0 },
    count_{ 0 }
{
    for (std::size_t i = 0; i < other.count_; ++i) {
        new (slot(i)) T(std::move(*other.slot(i)));
        ++count_;
    }
    for (std::size_t i = 0; i < other.count_; ++i) {
        other.slot(i)->~T();
    }
    other.count_ = 0;
    other.head_ = 0;
}

template <typename T, std::size_t N>
RingBuffer<T, N>::~RingBuffer()
{
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
        //Overwrite: destroy the oldest, advance head, and COMMIT the eviction now
        slot(0)->~T();
        head_ = (head_ + 1) % N;
        --count_;
    }
    //Construct first...
    new (slot(count_)) T(new_element);
    //...commit only after the construction succeeds
    ++count_;
}

template <typename T, std::size_t N>
void RingBuffer<T, N>::push_back(T&& new_element) {
    if (count_ == N) {
        slot(0)->~T();
        head_ = (head_ + 1) % N;
        --count_;
    }
    new (slot(count_)) T(std::move(new_element));
    ++count_;
}

template<typename T, std::size_t N>
template<typename ...Args>
T& RingBuffer<T, N>::emplace_back(Args && ...args)
{
    if (count_ == N) {
        slot(0)->~T();
        head_ = (head_ + 1) % N;
        --count_;
    }
    T* ptr{ new (slot(count_)) T(std::forward<Args>(args)...) };
    ++count_;
    return *ptr;
}

template <typename T, std::size_t N>
void RingBuffer<T, N>::pop_front() noexcept
{
    if (count_ == 0) return;
    slot(0)->~T();
    head_ = (head_ + 1) % N;
    --count_;
}

template <typename T, std::size_t N>
bool RingBuffer<T, N>::empty() const noexcept { return count_ == 0; }

template <typename T, std::size_t N>
bool RingBuffer<T, N>::full() const noexcept { return count_ == N; }

template <typename T, std::size_t N>
const T& RingBuffer<T, N>::front() const { assert(!empty()); return *slot(0); }

template <typename T, std::size_t N>
std::size_t RingBuffer<T, N>::size() const noexcept { return count_; }

#endif // RINGBUFFER_H
```

## Wrapping Up

We started this post with two latent bugs and a vague worry, and we leave it with a buffer whose every operation makes a guarantee we can name: keep `count_` equal to the number of live elements, and commit that change only after the step that can throw has finished. In `push_back` and `emplace_back` that meant reordering two statements; in the copy constructor it meant a `try`/`catch` that tears down its own partial work, because a constructor that throws gets no destructor to do it for them.

The most important idea to carry forward is the asymmetry between assignment and construction. An assignment operator runs on a live object and can trust that object's destructor to clean up after a throw; a constructor runs on an object that does not yet exist and must clean up by hand.

The overwrite path can offer only the basic guarantee, because making room destroys the oldest element irreversibly, and with every slot occupied there is nowhere to build the replacement first. That same shortage, nowhere to build first, is exactly what the next post is about.

## References
- [Exceptions](https://en.cppreference.com/w/cpp/language/exceptions) — cppreference
- [`throw` expression](https://en.cppreference.com/w/cpp/language/throw) — cppreference
- [`try`/`catch` block](https://en.cppreference.com/w/cpp/language/try_catch) — cppreference
- [`std::exception`](https://en.cppreference.com/w/cpp/error/exception) — cppreference
- [`std::runtime_error`](https://en.cppreference.com/w/cpp/error/runtime_error) — cppreference
- [`noexcept` specifier](https://en.cppreference.com/w/cpp/language/noexcept_spec) — cppreference
- [`std::terminate`](https://en.cppreference.com/w/cpp/error/terminate) — cppreference
- [RAII](https://en.cppreference.com/w/cpp/language/raii) — cppreference