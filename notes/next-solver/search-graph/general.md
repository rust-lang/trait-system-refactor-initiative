# The search graph

## `path_from_head`

We track the path from the highest cycle head to the provisional cache entry to only
use cache entries if they have the correct cycle kind.

- A
    - BA coinductive cycle
    - cB :x:
        - A inductive cycle

---

```rust
A :- B, C
B :- i, C
i :- C
C :- B, A
```
- A
    - B
        - iC¹
            - B inductive cycle
            - A inductive cycle
        - C²
            - B coinductive cycle
            - A coinductive cycle
    - C :exclamation: 
        - B
            - iC inductive cycle
            - C coinductive cycle
        - A coinductive cycle

It's sound to use the second provisional cache entry for `C` here.

---

```rust
A :- B, c
B :- c, A
c :- B
```
- A
    - B
        - c
            - B inductive cycle
        - A coinductive cycle
    - c :exclamation:
        - B
            - c inductive cycle
            - A inductive cycle

Here the cycles `ABA` and `AcBA` are different, so we must not reuse the cache entry.

---

```rust
A :- B, c
B :- c, D
c :- B
D :- A
```
- A
    - B
        - c
            - B inductive cycle
        - D
            - A coinductive cycle
    - c :exclamation: 
        - B
            - c inductive cycle
            - D
                - A inductive cycle

Again, `ABDA` and `AcBDA` are different, can't reuse.

---

```rust
A :- B, d
B :- c, A
c :- A, B
d :- B
```
- A
    - B
        - c
            - A inductive cycle
            - B inductive cycle
        - A coinductive cycle
    - dB :x: 
        - c :exclamation:
            - A inductive cycle
            - B inductive cycle
        - A inductive cycle

Here `ABA` and `AdBA` are different, so `B` cannot be reused. It's less clear whether `C` can
be reused. It could as long as the final provisional result of `B` is the same in both cycles.
We do not reuse it here for now.


## `nested_goals`

Tracking `nested_goals` is necessary for 2 reasons:
- we use the path from a goal to a cycle head when rebasing provisional cache entries
- we need to make sure we don't use global cache entries if a nested goal is on the stack
or the in the provisional cache. Test cases are tracked [separately](./why-nested-goals-global-cache.md).

We want to track as few `nested_goals` as possible, especially in cases where there are no
cycles. Tracking all nested goals has a significant negative perf impact. The idea is to
therefore only track nested goals which may actually end up as stack or provisional cache
entries when trying to access a global cache entry.

Due to overflow we could end up with applicable provisional cache entries for pretty much
all nested goals. However, "we need to track all nested goals for which a provisional cache
entry may apply", also means that "we need to only apply provisional cache entries for goals
which are guaranteed to be tracked in `nested_goals`".

Thinking about this is difficult as whether we should apply provisional cache entries
is decided before we try to evaluate a goal while we only move a goal to the global cache
after evaluating it. We need to decide whether a provisional result would apply for any
nested goal using the state of `nested_goals` after evaluating the whole goal.

The current approach is to not apply provisional results if they `encountered_overflow`
unless the current goal also already encountered overflow. This is correct as we always
track all nested goals once we've encountered overflow, however, it's likely far from
the best solution here.

## fun general tests

Whether a global cache entry is used must not impact the provisional cache.

- A
    - B
        - CB cycle
        - A cycle
    - F
        - CD provisional cache hit
    - eFCBC inductive path, provisional entry of `F` is not applicable
    - F should be a provisional cache hit again

---

Bailing on ambiguity must taint or remove provisional cache entries

```rust
#![feature(rustc_attrs, marker_trait_attr)]

#[marker]
trait A {}
impl<T: B> A for T {}
impl<T: C> A for T {}

#[rustc_coinductive]
trait B {}
impl<T: C + A> B for T {}

#[marker]
#[rustc_coinductive]
trait C {}
impl<T: B> C for T {}

fn is_a<T: A>() {}
fn main() {
    is_a::<u32>();
}
```
- `u32: A`
    - impl1
        - `u32: B`
            - `u32: C`
                - `u32: B` coinductive cycle
        - `u32: A` inductive cycle
        - final result ambig, no fixpoint -> bail
  - impl2
    - `u32: C` must also be ambig as it depends on B :skull: 

---

Tainting on ambiguity causes unavoidable undesirable ambiguity.
```rust
#![feature(rustc_attrs, marker_trait_attr)]

#[marker]
trait A {}
impl<T: B> A for T {}
impl<T: C> A for T {}

#[rustc_coinductive]
trait B {}
impl<T: C + A> B for T {}

#[marker]
#[rustc_coinductive]
trait C {}
impl<T: B> C for T {}
impl<T> C for T {}

fn is_a<T: A>() {}
fn main() {
    is_a::<u32>();
}
```
- `u32: A`
    - impl1
        - `u32: B`
            - `u32: C`
                - impl1
                    - `u32: B` coinductive cycle
                - impl2 -> trivially ok
        - `u32: A` inductive cycle
        - final result ambig, no fixpoint -> bail
  - impl2
    - `u32: C` doesn't depend on B, should ideally be ok :skull: 