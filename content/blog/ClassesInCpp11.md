+++
title = "Ring Buffer Series Part 11 — Exception Safety II: Strong Guarantee & Copy-and-swap"
date = "2026-06-23T12:00:00+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "classes"]

[images]
    featured_image = "/img/blog/ringbuffer11.jpg"
+++

![ringbuffer11](/img/blog/ringbuffer11.jpg)
<br>In [part 10](ClassesInCpp10) we had a buffer whose every operation made a guarantee we could name. Most of them were as strong as we could want. Two were not, and we left ourselves a note about both. Our copy-assignment was only basic, and worse, it was destructive on failure: it tore down the buffer's current contents before it started rebuilding, so a throw partway through the rebuild lost the old contents and the new ones at once. And our overwrite path was basic for a reason we could state precisely: making room destroys the oldest element irreversibly, and with every slot occupied there is nowhere to build the replacement first.

Remember that in [part 9](ClassesInCpp9), when we first wrote copy-assignment by hand, we talked about the copy-and-swap idiom which is the standard way to write assignment? Remember that we didn't use it because the version everyone reaches for swaps the raw bytes of our inline storage, and that is undefined behavior. We said copy-and-swap would return. This is where it returns and it proposes an interesting idea for the RingBuffer: the strong guarantee means doing everything that can throw first, and only then committing with operations that cannot throw. On our inline storage, that "cannot throw" commit is available only when `T`'s move is `noexcept`. Let's get to it!

## The Strong-Guarantee Recipe

Strip away the buffer for a moment and the recipe for the strong guarantee is always the same three steps:

1. Do all the work that might throw against a fresh, separate object, off to the side, where a failure cannot touch the thing we are modifying.
2. If that work succeeds, commit it into place using operations that cannot throw.
3. Nothing the caller can observe changes until step 2, so if step 1 throws, it is as if the call never happened.

The hard part is almost never step 1. It is step 2: finding a commit that genuinely cannot throw. A commit that can fail halfway leaves us exactly where the destructive copy-assignment left us, having already disturbed the object with no way back. So the real question for any container is: what is my no-throw commit?

## But std::vector has it, Why Can't We?

For a heap-backed container like `std::vector`, the answer is almost magically cheap. A vector that needs to grow allocates a new, bigger block, builds the new contents over there, and then commits by writing a single pointer: it points its internal `data_` at the new block and frees the old one. Assigning a pointer cannot throw. The elements themselves never move during the commit; ownership of a whole block changes hands in one instruction.

Our buffer cannot do this, and the reason is the same one that has shaped every hard decision in this series. Our storage is an `alignas(T) std::byte buffer_[N * sizeof(T)]` baked directly **inside** the object. There is no pointer to hand over. The bytes are part of the `RingBuffer` itself. So a commit for us cannot be a pointer swap; it has to physically move each element from the temporary into our storage, one at a time, by calling `T`'s move constructor. And moving is only guaranteed not to throw when `T`'s move constructor is `noexcept`.

![HeapVsInline](/img/blog/HeapVsInline.jpg)

A vector commits by relabelling a pointer; we commit by moving elements. One is unconditionally safe; the other is only as safe as `T`'s move.

## Copy-and-swap Returns!

The copy and swap idiom always bundles together two things: its mechanics and its sequencing. The key for us to implement the idiom is to be able to keep these two things separate.

The sequencing is the strong-guarantee recipe itself: make a complete copy first, then commit by swapping. Nothing the caller sees changes until the copy has already succeeded. That part was always sound, and it is exactly what we want.

The mechanics are what we said not to in part 9. The textbook copy-and-swap swaps each member with `std::swap`, and for our buffer that meant `std::swap(buffer_, other.buffer_)` blitting the raw bytes of two `std::byte` arrays. That swaps the bytes of objects whose constructors never ran on the destination, without ever calling a move constructor or destructor. For a `T` with self-referential state or any non-trivial move, the result is undefined behavior.
![UBSwap](/img/blog/UBSwap.jpg)

## A Simple std::string Demo

Most standard libraries give short strings the small-string optimization: rather than allocate on the heap, a short string keeps its characters in a small buffer inside the `string` object, and its internal data pointer points at that inline buffer, so a pointer into itself, which is what we need to see the above's diagram in action. So let's place two short strings in raw storage, `std::swap` the bytes in the common way that we didn't use, and watch what happens:
```cpp
#include <iostream>
#include <string>
#include <cstddef>
#include <new>
#include <utility>
#include <memory>   // std::destroy_at

// Is pointer p inside the raw storage region 'slot'?
static bool in_slot(const void* p, const std::byte* slot) {
    return p >= static_cast<const void*>(slot)
        && p <  static_cast<const void*>(slot + sizeof(std::string));
}

int main() {
    alignas(std::string) std::byte slotA[sizeof(std::string)];
    alignas(std::string) std::byte slotB[sizeof(std::string)];

    std::string* a = new (slotA) std::string("hi");   // short -> small-string optimization
    std::string* b = new (slotB) std::string("yo");

    std::cout << "before the swap:\n" << std::boolalpha;
    std::cout << "  a = \"" << *a << "\"   a.data() lives in slotA: " << in_slot(a->data(), slotA) << "\n";
    std::cout << "  b = \"" << *b << "\"   b.data() lives in slotB: " << in_slot(b->data(), slotB) << "\n";

    std::swap(slotA, slotB);   // the part 9 mistake: blit the raw bytes, no constructor runs

    std::cout << "\nafter std::swap(slotA, slotB) on the raw bytes:\n";
    std::cout << "  a = \"" << *a << "\"   a.data() in slotA: " << in_slot(a->data(), slotA)
              << " , in slotB: " << in_slot(a->data(), slotB) << "\n";
    std::cout << "  b = \"" << *b << "\"   b.data() in slotB: " << in_slot(b->data(), slotB)
              << " , in slotA: " << in_slot(b->data(), slotA) << "\n";

    std::cout << "\ncleaning up (running ~string on each)..." << std::endl;
    std::destroy_at(a);
    std::destroy_at(b);
    std::cout << "done\n";
}
```
And here is the output
```
before the swap:
  a = "hi"   a.data() lives in slotA: true
  b = "yo"   b.data() lives in slotB: true

after std::swap(slotA, slotB) on the raw bytes:
  a = "hi"   a.data() in slotA: false , in slotB: true
  b = "yo"   b.data() in slotB: false , in slotA: true

cleaning up (running ~string on each)...
munmap_chunk(): invalid pointer
Aborted
```

Read it slowly, because two separate things went wrong. First, the values look unchanged, `a` is still `"hi"`, `b` is still `"yo"`, even though we just "swapped" them. That is the crossed pointers from the diagram: `a`'s data pointer now points into `slotB`, and `slotB`'s bytes happen to hold the characters we want, so `a` reads its original value through a pointer aimed at the wrong object. The `in slotA: false , in slotB: true` line is the dangling pointer caught red-handed.

Then it dies. When `~string()` runs it decides whether to free heap memory by checking whether the data pointer still points at the object's own inline buffer. After the byte swap it does not, `a`'s pointer points into `slotB`, so the destructor concludes the string is heap-allocated and calls the equivalent of `delete` on a stack address. Hence `munmap_chunk(): invalid pointer` and the abort.

This is undefined behavior, so the exact output is specific to this compiler and library; a different implementation might appear to work, corrupt silently, or crash some other way, which is the worst property a bug can have. The single root cause is that `std::swap` on the bytes never called `std::string`'s move constructor, and that move constructor is the only operation that knows to re-point the inline pointer at its new home. The bytes alone cannot.

In part 9 we discovered what a correct swap would move the elements one at a time. We can write that, and we already have all the pieces. A swap built out of our move constructor and our move assignment is correct, and its `noexcept`-ness automatically mirrors `T`'s move, because that is all it calls:

```cpp{hl_lines=["9-17"]}
template <typename T, std::size_t N>
class RingBuffer
{
public:
    // Previous code remains the same.
    
    const_iterator cbegin() const { return begin(); }
    const_iterator cend() const { return end(); }
    // A correct, hand-written swap: NOT std::swap on the byte array.
    // Built from the move constructor and move assignment, so its noexcept mirrors T's move.
    friend void swap(RingBuffer& a, RingBuffer& b)
    noexcept(std::is_nothrow_move_constructible_v<T>)
    {
    RingBuffer tmp(std::move(a));
    a = std::move(b);
    b = std::move(tmp);
    }

    RingBuffer();
    RingBuffer(const RingBuffer& other);
    RingBuffer(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>);

    //Rest of the code remains the same

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
```

This is similar to how the default `std::swap` works. Move one into a temporary, move the second into the first, move the temporary into the second. Three moves. This is what swapping is for any type that has not specialized it into something cheaper. The only types that get a one-line pointer-exchange swap are the ones, like `vector`, that hold a pointer to swap. We hold our elements inline, so for us swapping genuinely means moving.

```cpp
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

int main() {
    RingBuffer<Throwing, 4> a; a.emplace_back(1); a.emplace_back(2);
    RingBuffer<Throwing, 4> b; b.emplace_back(7); b.emplace_back(8); b.emplace_back(9);
    swap(a, b);
}
```

The output (the `Throwing` instrument from part 10 prints every special operation; `live` is its leak counter):

```
 [Ctro] 1  (live=1)
 [Ctro] 2  (live=2)
 [Ctro] 7  (live=3)
 [Ctro] 8  (live=4)
 [Ctro] 9  (live=5)
 before: a = [ 1 2 ]
         b = [ 7 8 9 ]
 -- swap(a, b) --
 [MOVE] 1  (live=6)
 [MOVE] 2  (live=7)
 [Dtor] 1  (live=6)
 [Dtor] 2  (live=5)
 [MOVE] 7  (live=6)
 [MOVE] 8  (live=7)
 [MOVE] 9  (live=8)
 [Dtor] 7  (live=7)
 [Dtor] 8  (live=6)
 [Dtor] 9  (live=5)
 [MOVE] 1  (live=6)
 [MOVE] 2  (live=7)
 [Dtor] 1  (live=6)
 [Dtor] 2  (live=5)
 after:  a = [ 7 8 9 ]
         b = [ 1 2 ]
 -- end of scope --
```

Read the three movements in the trace. First `a`'s elements (`1`, `2`) move into the temporary, and `a`'s husks are destroyed. Then `b`'s elements (`7`, `8`, `9`) move into `a`, and `b`'s husks are destroyed. Finally the temporary's elements (the original `1`, `2`) move into `b`. Three moves, exactly as the code says, and `live` returns to where it started: no leak, no over-destruction. The swap is correct.

## Fixing Our Assignment

Now we can pay the first part 10 debt. We want an assignment that, if it throws, leaves `*this` exactly as it was. We now know how to do it! Copy `other` into a fresh buffer first, and only then swap that buffer into place. Consider the following:

```cpp
template <typename T, std::size_t N>
void cas_assign(RingBuffer<T, N>& lhs, const RingBuffer<T, N>& rhs) {
    RingBuffer<T, N> tmp(rhs);   // the only fallible work; lhs is not touched yet
    swap(lhs, tmp);              // commit; tmp carries lhs's old contents away and destroys them
}
```

The copy into `tmp` is the only step that can throw, and it happens entirely off to the side. If it throws, we never reach the `swap`, and `lhs` is untouched. If it succeeds, the `swap` commits the new contents into `lhs` and hands `lhs`'s old contents to `tmp`, which destroys them when it goes out of scope.

Let's prove it. We arm `Throwing` so that copying the source throws partway through, and check what `lhs` looks like afterward.

```cpp
int main() {
    Throwing::arm(7);                          // the 7th construction attempt throws
    RingBuffer<Throwing, 4> lhs; lhs.emplace_back(10); lhs.emplace_back(11);
    RingBuffer<Throwing, 4> rhs; rhs.emplace_back(20); rhs.emplace_back(21); rhs.emplace_back(22);
    // lhs before: [ 10 11 ]
    try {
        cas_assign(lhs, rhs);                  // copies rhs into tmp; the copy throws
    } catch (const std::exception& e) {
        std::cout << " caught: " << e.what() << '\n';
    }
    // lhs after?
}
```

The output:

```
 [Ctro] 10  (live=1)
 [Ctro] 11  (live=2)
 [Ctro] 20  (live=3)
 [Ctro] 21  (live=4)
 [Ctro] 22  (live=5)
 lhs before: [ 10 11 ]
 -- cas_assign(lhs, rhs); the copy will throw --
 [COPY] 20  (live=6)
 [THROW] on construction attempt #7
 [Dtor] 20  (live=5)
 caught: Throwing failed
 lhs after:  [ 10 11 ]
 (should be 10 11)
```

There it is. The copy of `rhs` into `tmp` got one element in (`[COPY] 20`) and then threw on the next. The `[Dtor] 20` right after the throw is `tmp`'s own copy constructor cleaning up its partial batch. The exact hand-rollback we built in part 10. The exception propagated out before `swap` was ever called, and `lhs` is still `[ 10 11 ]`, byte for byte what it was. This is the strong guarantee, the operation either succeeds or it is as if it never happened.

![CopyAndSwapStrong](/img/blog/CopyAndSwap.jpg)

Knowing that it works, we can actually update our assignment operator, consider the following:

```cpp
RingBuffer& operator=(RingBuffer other)          // 'other' is the caller's copy
    noexcept(std::is_nothrow_move_constructible_v<T>)
{
    swap(*this, other);
    return *this;
}
```

Because the parameter is taken by value, the compiler makes the copy for us using the copy constructor for an lvalue argument, the move constructor for an rvalue, so this single operator serves as both copy- and move-assignment. The fallible copy happens in forming `other`, before the body runs; the body only swaps.

## But Wait, There is More!

But notice the `noexcept` clause on that operator, and on `swap`. They are not unconditional. They are `noexcept` only when `T`'s move is. We have been quietly assuming the commit cannot throw, so what happens when it can?

To find out we need a different instrument. Our `Throwing` from part 10 has a `noexcept` move on purpose, so it can never break the commit. We need a type whose move can throw on demand. Consider the following:

```cpp
// Move is NOT noexcept and can be armed to throw on the Nth MOVE. Copy never throws.
struct ThrowingMove {
    int id;
    static inline int live          = 0;
    static inline int moved         = 0;
    static inline int throw_move_at = -1;
    static void arm_move(int nth) { live = 0; moved = 0; throw_move_at = nth; }

    explicit ThrowingMove(int i) : id{ i } { ++live; std::cout << " [Ctro] " << id << "  (live=" << live << ")\n"; }
    ThrowingMove(const ThrowingMove& o) : id{ o.id } { ++live; std::cout << " [COPY] " << id << "  (live=" << live << ")\n"; }
    ThrowingMove(ThrowingMove&& o) : id{ o.id } {                 // note: no noexcept
        if (++moved == throw_move_at) {
            std::cout << " [THROW] on move attempt #" << moved << '\n';
            throw std::runtime_error("move failed");
        }
        ++live; std::cout << " [MOVE] " << id << "  (live=" << live << ")\n";
    }
    ~ThrowingMove() { --live; std::cout << " [Dtor] " << id << "  (live=" << live << ")\n"; }
};
```

The compiler already sees the difference. With our two instruments, the buffer's `swap` inherits each type's move guarantee exactly:

```
is_nothrow_move_constructible_v<Throwing>     = true
is_nothrow_move_constructible_v<ThrowingMove> = false
swap(RingBuffer<Throwing,4>)     is noexcept  = true
swap(RingBuffer<ThrowingMove,4>) is noexcept  = false
```

Now run the same assignment, with `ThrowingMove`, and arm a *move* to throw during the swap:

```cpp
int main() {
    ThrowingMove::arm_move(3);                 // the 3rd move throws
    RingBuffer<ThrowingMove, 4> lhs; lhs.emplace_back(10); lhs.emplace_back(11);
    RingBuffer<ThrowingMove, 4> rhs; rhs.emplace_back(20); rhs.emplace_back(21); rhs.emplace_back(22);
    // lhs before: [ 10 11 ]
    try {
        cas_assign(lhs, rhs);                  // copy succeeds; a move inside swap throws
    } catch (const std::exception& e) {
        std::cout << " caught: " << e.what() << '\n';
    }
    // lhs after?
}
```

The output:

```
 [Ctro] 10  (live=1)
 [Ctro] 11  (live=2)
 [Ctro] 20  (live=3)
 [Ctro] 21  (live=4)
 [Ctro] 22  (live=5)
 lhs before: [ 10 11 ]
 -- cas_assign(lhs, rhs); a move during the swap will throw --
 [COPY] 20  (live=6)
 [COPY] 21  (live=7)
 [COPY] 22  (live=8)
 [MOVE] 10  (live=9)
 [MOVE] 11  (live=10)
 [Dtor] 10  (live=9)
 [Dtor] 11  (live=8)
 [THROW] on move attempt #3
 [Dtor] 10  (live=7)
 [Dtor] 11  (live=6)
 [Dtor] 20  (live=5)
 [Dtor] 21  (live=4)
 [Dtor] 22  (live=3)
 caught: move failed
 lhs after:  [ ]
 live = 3 (only rhs's 3 should remain)
```

The copy of `rhs` into `tmp` succeeds completely (`[COPY] 20 21 22`). Then `swap` begins. Its first step moves `lhs`'s `10` and `11` into the swap's internal temporary and destroys `lhs`'s husks, and right there `lhs` has been emptied. Its second step starts moving the new contents into `lhs`, and on the very first move it throws. The exception unwinds: the swap's temporary (holding the old `10`, `11`) is destroyed, and `tmp` (holding `20 21 22`) is destroyed.

When the dust settles, `lhs` is `[ ]`. Not `[ 10 11 ]`, not `[ 20 21 22 ]`, empty. The old contents are gone, the new ones never arrived. `live` returns to `3` (just `rhs`), so there is no leak and `lhs` is a perfectly valid empty buffer. This is the basic guarantee, not the strong one. The commit disturbed `lhs` and then failed with no way back, because a throwing move cannot be cleanly undone.

So copy-and-swap on our inline storage is strong if and only if the swap cannot throw, which is if and only if `T`'s move is `noexcept`. The sequencing was never the problem; the commit is.

### Why we keep the explicit operators

Given that the by-value operator is so tidy, why not ship it as our assignment? Two reasons:

Our elegant three-line `swap` is built on move-assignment (`a = std::move(b)`). If we make the by-value `operator=(RingBuffer)` our only assignment, then that `a = std::move(b)` inside `swap` now resolves to the by-value operator, whose body calls `swap`, which calls the operator, which calls `swap`. It is infinite recursion, and it compiles without complaint and then overflows the stack at runtime (a segmentation fault). To break the cycle, the copy-and-swap operator needs a `swap` that does not route through any assignment operator: one that moves elements directly with placement-new and explicit destruction. That is more machinery than the three-liner, written solely to support an operator we have other reasons to avoid.

![RecursionTrap](/img/blog/RecursionTrap.jpg)

The second reason is the conditional we just demonstrated. A "strong" copy-assignment is strong only when `T`'s move is `noexcept`; for any other `T` it silently degrades to the basic, destructive behavior of the demo above. Advertising it as strong would be a lie for exactly the types that need the promise most. We will come back to this in a later post, for now we keep the `swap` itself as it is correct and useful in its own right.

## Why `std::move_if_noexcept` Can't Help Us Much

`std::move_if_noexcept`  looks at `T`'s move constructor and returns an rvalue reference when the move is `noexcept` (so the element is moved, cheaply), but a `const` lvalue reference when the move can throw (so the element is copied instead). "Move if it's safe, otherwise copy." It is precisely how `std::vector` keeps reallocation strong: when a vector grows, it relocates with `move_if_noexcept`, so a type with a throwing move is copied into the new block rather than moved, and if a copy throws, the new block is discarded and the old block is still completely intact.

![RelocationThrow](/img/blog/RelocationThrow.jpg)

That last clause is why it works for `vector` and why it cannot rescue us. The vector relocates into separate storage and leaves the source untouched until every element has arrived. Its escape hatch depends entirely on having somewhere else to build. Our buffer has one inline storage region and nowhere else. To commit, we must overwrite `*this`'s storage, which means destroying `*this`'s current contents first, and once we have done that, choosing "copy instead of move" no longer helps, because the thing we would roll back to is already gone. `move_if_noexcept` answers "move or copy?"; it cannot answer "where do I build so the original survives?", and on fixed inline storage that is the question that matters. We will also come back to this in the future.

## The Full Buffer's Missing Spare Slot

When the buffer is full and we `push_back`, we must destroy the oldest element to make room, and then construct the new one in the freed slot. There is no spare slot to build the replacement in first, so the destruction of the old element is an irreversible commit that happens before the fallible construction. If that construction throws, the old element is already gone.

Stated as the recipe, the overwrite path fails step 1: it cannot do the fallible work against a separate object, because it has no separate object to work against. The fixes are exactly the two escapes the recipe predicts:

- Find a spare. Keep one extra slot of capacity (or a small side buffer) so the replacement can be built before the old element is destroyed, but this changes the data structure and our fixed `N` cannot do this without redefining what `N` means.
- Lean on a `noexcept` move. If `T`'s move cannot throw, build the replacement in a temporary, then destroy the old element and move the replacement in, the move-in being the no-throw commit. The shape is the same copy-then-commit we used for assignment, and it carries the same condition: strong only when `T`'s move is `noexcept`.

## When a `noexcept` Move Lies

We keep saying the commit must use operations that cannot throw, and we enforce that with `noexcept`. But `noexcept` is a promise, not a force field. What happens if a move constructor is marked `noexcept` and throws anyway?

The runtime calls `std::terminate`, and the program dies. There is no unwinding, no `catch`, no recovery, an exception escaping a `noexcept` function is a hard stop. Consider the following:

```cpp
struct Liar {
    int id;
    explicit Liar(int i) : id{ i } { std::cout << " [Ctro] " << id << std::endl; }
    Liar(const Liar&) = default;
    Liar(Liar&& o) noexcept : id{ o.id } {                 // the lie: declared noexcept, throws anyway
        std::cout << " [MOVE] about to throw from a noexcept move..." << std::endl;  // endl flushes
        throw std::runtime_error("boom");
    }
    ~Liar() { std::cout << " [Dtor] " << id << std::endl; }
};

int main() {
    RingBuffer<Liar, 4> a;
    a.emplace_back(1);
    std::cout << "-- moving the buffer; its move is noexcept and moves each Liar --" << std::endl;
    RingBuffer<Liar, 4> b{ std::move(a) };       // noexcept buffer move -> Liar's lying move throws -> terminate
    std::cout << "this line is never reached" << std::endl;
}
```

Because `Liar`'s move is declared `noexcept`, our buffer's move is `noexcept` too, so when it moves the one `Liar` and that move throws, the exception is trying to escape a `noexcept` function. The output:

```
 [Ctro] 1
-- moving the buffer; its move is noexcept and moves each Liar --
 [MOVE] about to throw from a noexcept move...
terminate called after throwing an instance of 'std::runtime_error'
  what():  boom
Aborted
```

Note the `std::endl` in `Liar`'s prints. `std::terminate` aborts the process without running the normal flush, so a buffered `std::cout` would lose its last lines and the instrumentation would simply vanish; flushing explicitly is the only reason we see the `[MOVE]` line at all. The conditional `noexcept` on our move is what lets the standard library, and our own `swap`, trust the commit.

## Our Buffer's Guarantees

Updating the Guarantees Table previously introduced:

| Operation | Guarantee | Why |
| --- | --- | --- |
| `size`, `empty`, `full`, `pop_front` | no-throw | marked `noexcept`; touch only `count_`/`head_` |
| move ctor / move-assign | no-throw *when `T`'s move is* | conditional `noexcept` from part 9 |
| `swap` | no-throw *when `T`'s move is* | three moves; inherits `T`'s move guarantee |
| `push_back` / `emplace_back`, not full | strong | construct first, commit `count_` after |
| `push_back` / `emplace_back`, overwriting | basic | the evicted element cannot be recovered |
| copy constructor | basic | `try`/`catch` cleans up the partial batch |
| copy-assignment (shipped) | basic | destroys current contents, then rebuilds |
| copy-assignment via copy-and-swap | strong when `T`'s move is, else basic | copy off to the side, commit by `swap` |

## No Such Thing as a Free Lunch

The strong assignment costs a transient second buffer's worth of storage, for a moment we hold both the old contents and a full copy of the new ones, and it costs a `noexcept` move on `T`, without which it quietly falls back to basic. The basic assignment costs neither: no extra storage, no constraint on `T`. For a buffer of small, cheap, nothrow-movable elements, the strong version is a clear win and the by-value operator is a fine way to get it. For a buffer of large elements where the transient copy is expensive, or for a `T` whose move can throw, the basic guarantee may be the better engineering choice. Naming the guarantee is what lets us make that call deliberately instead of by accident.

## The Code

Here is the updated buffer:

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

    RingBuffer& operator=(const RingBuffer& other)   // Copy Assignment - basic guarantee
    {
        if (this == &other) return *this;
        for (std::size_t i = 0; i < count_; ++i) {
            slot(i)->~T();
        }
        head_ = 0;
        count_ = 0;
        for (std::size_t i = 0; i < other.count_; ++i) {
            new (slot(i)) T(*other.slot(i));
            ++count_;
        }
        return *this;
    }

    RingBuffer& operator=(RingBuffer&& other) noexcept(std::is_nothrow_move_constructible_v<T>)  // Move Assignment
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

    // A correct, hand-written swap: NOT std::swap on the byte array (that was the part 9 UB).
    // Built from the move constructor and move assignment, so its noexcept mirrors T's move.
    friend void swap(RingBuffer& a, RingBuffer& b)
        noexcept(std::is_nothrow_move_constructible_v<T>)
    {
        RingBuffer tmp(std::move(a));
        a = std::move(b);
        b = std::move(tmp);
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
    try {
        for (; count_ < other.count_; ++count_) {
            new (slot(count_)) T(*other.slot(count_));
        }
    }
    catch (...) {
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
        slot(0)->~T();
        head_ = (head_ + 1) % N;
        --count_;
    }
    new (slot(count_)) T(new_element);
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

The strong guarantee is a recipe: do the fallible work against a separate object, then commit with operations that cannot throw. For `std::vector` the commit is a pointer swap and is unconditionally free; for our inline buffer the commit is moving elements, so it is free only when `T`'s move is `noexcept`. Copy-and-swap is implemented by splitting it in two, its sequencing (copy first, then swap) was always right, and its mechanics are right too once `swap` moves elements one at a time instead of blitting bytes. That `swap` now ships, and a copy-and-swap assignment built on it is strong exactly when `T`'s move is nothrow, and basic otherwise. We keep the explicit basic operators, because the strong version carries a condition the class does not yet impose on `T`, and because routing assignment through `swap` while building `swap` on assignment is an infinite-recursion trap.

`std::move_if_noexcept` is the concept that names the whole nothrow-move thread, but it rescues a heap-backed reallocation precisely because that reallocation builds in separate storage. the one thing our fixed inline buffer does not have. That observation is also the answer to the overwrite path: it fails for the same reason, nowhere to build first, and a real fix wants the spare room and we will come back to this in the future.

Next, we step back from exception safety and make the buffer feel finished: `back()`, `clear()`, `capacity()`, a bounds-checked `at()` that throws, reverse iterators (`rbegin`/`rend`), and container comparison with `operator==` and `operator<=>`.

## References

- [Exceptions](https://en.cppreference.com/w/cpp/language/exceptions) — exception handling and the exception-safety guarantees.
- [`noexcept` specifier](https://en.cppreference.com/w/cpp/language/noexcept_spec) — declaring (and conditionally declaring) that a function does not throw.
- [`std::is_nothrow_move_constructible`](https://en.cppreference.com/w/cpp/types/is_move_constructible) — the trait our `swap` and moves are conditioned on.
- [`std::move_if_noexcept`](https://en.cppreference.com/w/cpp/utility/move_if_noexcept) — move when the move is nothrow, otherwise copy.
- [`std::swap`](https://en.cppreference.com/w/cpp/algorithm/swap) — the customization point our hidden friend hooks into.
- [Copy assignment operator](https://en.cppreference.com/w/cpp/language/copy_assignment) — including the copy-and-swap form.
- [`std::terminate`](https://en.cppreference.com/w/cpp/error/terminate) — what happens when an exception escapes a `noexcept` function.
- [placement `new`](https://en.cppreference.com/w/cpp/language/new#Placement_new) — constructing into our raw storage, carried from part 6.