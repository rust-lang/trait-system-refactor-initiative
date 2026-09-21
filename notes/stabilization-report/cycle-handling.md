# Canonicalization, caching, and cycle handling

## Old-style canonicalization

We still use the old style canonicalization in some places, especially if the code is shared by both trait solvers. We should remove one of them soon after stabilization.

We should remove one of them soon after stabilization https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/canonical/query_response.rs#L92-L108
We still 

## Erasing universe information in query inputs

that's cool stuff, TODO
- better for perf
- does not matter
- means that in https://github.com/rust-lang/rust/issues/161404 we don't trigger https://github.com/rust-lang/rust/blob/aea4dd4b0377fb5881542815dc3c2352394e8514/compiler/rustc_infer/src/infer/relate/generalize.rs#L425

## Caching the unconstrained inference variables of normalization

Minor breakage and jank https://github.com/rust-lang/trait-system-refactor-initiative/issues/215

https://github.com/rust-lang/trait-system-refactor-initiative/issues/275


## Rerunning canonical goals until reaching a fixpoint

This causes some minor breakage
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/209
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/118

## Cycle handling

The next-generation trait solver handles cycles differently than the old solver. This change is necessary due to https://github.com/rust-lang/trait-system-refactor-initiative/issues/10. The old trait solver did not track cycle participants sufficiently.

A cycle is now considered coinductive if at least one step is productive. In the old cycles were only coinductive if all goals involved in the cycle were coinductive. Importantly, whether a cycle is coinductive does not depend on the goals in the cycle, but the steps between goals; the reason why were proving nested goals: [source](https://github.com/rust-lang/rust/blob/28b5293debd90e0ad9b8ccb937c03e499cfc2170/compiler/rustc_next_trait_solver/src/solve/eval_ctxt/mod.rs#L430-L469).

We now not only have coinductive and inductive cycles, but also ambiguous cycles. This is necessary because we need to tr[source](https://github.com/rust-lang/rust/blob/28b5293debd90e0ad9b8ccb937c03e499cfc2170/compiler/rustc_type_ir/src/search_graph/mod.rs#L115-L158).
- explicitly 3 different cycle kinds, why NoSolution, why ambig, why yes

https://github.com/rust-lang/rust/issues/150508

### Allows more code to compile

https://github.com/rust-lang/trait-system-refactor-initiative/issues/114#issuecomment-3073743088

```rust
trait Trait {
    type Assoc;
}

fn foo<T: Trait<Assoc = <T as Trait>::Assoc>>(_: T::Assoc) {}
```

### Breaking change

In the old solver non-productive cycles are always ambiguous in `evaluate`. Tthe old solver only uses `evaluate` to select candidates and then processes these candidates in `fulfill`. This means `fulfill` also needs to handle cycles. We currently treating cycles in fulfill as an error, which can impact method selection. https://github.com/rust-lang/trait-system-refactor-initiative/issues/224

### Weird jank

We still haven't fully figured out the way cycle handling should work.

On a conceptional level, normalizing aliases in a goal should happen *outside* of that goal. This happens accidentally if we eagerly normalize. However, we can't always eagerly normalize higher-ranked aliases, so these may get normalized inside of the goal. This may change the cycle kind and I can't tell whether this can cause any issues. I think that for now things are fine here.
we normalize inside of goals if there are hr aliases, does this mess us cycle detection?

vibe: negative reasoning and solver cycles
https://github.com/rust-lang/trait-system-refactor-initiative/issues/122