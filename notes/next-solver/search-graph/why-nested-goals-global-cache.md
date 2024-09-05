# nested goals of global cache entries

These are tracked to disable the global cache in cases where reevaluating the goal ends up
being stack-dependent. This can happen by either ending up with a cycle on the stack or by
depending on a provisional cache entry.

We have to disable a cache entry if it's the root of a cycle and another cycle participant
is already on the stack.

- ABA cycle
- BA :x:

As the root of a cycle can change its result, we have to also be careful with global cache
entries which depend on a cycle, even if they aren't directly involved.

- ABCB cycle
- CB
    - C cycle
    - A :x:

## provisional cache entries

Once a goal depends on a provisional cache entry, its behavior could widely differ from
previous executions.

- ABCB cycle
- D
    - C
        - BC cycle
        - D cycle!
    - A
        - B
            - C provisional cache hit
        - iA :x:
 
Here `A` starts to depend on `i` due to the provisional cache hit, causing it to now depend
on itself. This means the entry for `A` must not be used. The outer `A` has to be disabled as
`C` is in the provisional cache. However, this does not disable the inner `A` by itself. Unlike
`DCD` and `DABCD`, the cycle `DAiABCD` is inductive.

---

Simply checking whether a goal is already on the stack before using its global cache entry is not
enough.

- ABCDC cycle
- E
    - D
        - CD cycle
        - E cycle!
    - BCD provisional cache hit
    - A :x:
        - B provisional cache hit

---

- aBCDC cycle
- E
    - D
        - CD cycle
        - E cycle
    - B
        - CD provisional cache hit
        - a :x:

The global cache entry for `a` must be disabled here as `B` is on the stack. The provisional cache entries for `C` and `D` are not applicable as cycles involving `a` are inductive.

## overflow


- ABB overflow
- BB :question: equal depth as global cache entry but cycle

---

- A
    - BA overflow
    - B should use cache

---

Whether evaluating a goal encounters overflow can result in cycles, completely changing its
evaluation.

- ABCC overflow
- B
    - CC cycle
    - B :question: equal depth as global cache entry but cycle 

---

- ABCDC overflow
- C
    - DC cycle
    - B :x: global cache hit, but C is on the stack

---

- ABC
- D
    - EA
        - BC overflow
        - D cycle
    - A :x: both the global cache and the provisional cache are applicable?

---

Simply checking whether a goal itself is in the provisional cache is not enough.

- ABCD
- E
    - FGB
        - CD overflow
        - E cycle
    - A :question: B is in the provisional cache?

---

This test failed with an impl where we only considered provisional cache entries
if a goal is a cycle participant.

- A
    - B
        - CA overflow
        - A cycle
    - D
        - C global cache hit
        - BC overflow
        - B :question: