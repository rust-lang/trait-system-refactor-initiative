# Stash

A temporary stash of things worthy of documentation found while working towards the stabilization.


## New `FulfillmentContext`

## Reimplementing `select`

Selection is implemented separately from trait solving in the new solver. TODO WHY?

This means trait solving and selection can differ in the way they handle candidate preference. Trait solving only merges user-written impl and the builtin trait object impl candidates if they have the same constraints, while selection needs to always prefer the builtin trait object impl.

This means we cannot use these two interchangeably https://github.com/rust-lang/trait-system-refactor-initiative/issues/241

THis feels outdated, do looky look :>

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/solve/select.rs#L20

## Reimplementing rustdoc auto-trait impl generation

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/auto_trait.rs#L84-L91

## Const generics

uwu https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/mod.rs#L460

## When do we use `evaluate_obligation`

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/query/evaluate_obligation.rs#L86

## Candidate preference

We now merge where-clauses by checking the constraits in their query response instead of a syntactic check. TODO: does this result in behavior differences. TODO: YES no constraints + MAYBE sus https://rust-lang.zulipchat.com/#narrow/channel/144729-t-types/topic/resolving.20equal.20regions/near/623504310

The candidate preference rules for `Trait` goals are the same as with the old solver since the FCP back in https://github.com/rust-lang/rust/pull/132325.

This is not the case for `Projection` goals. The old solver prefers builtin trait object candidates over user-written impls while the new solver does not, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/101.

We also still intentionally prefer builtin trait-object candidates over impls to avoid breakage: https://github.com/rust-lang/trait-system-refactor-initiative/issues/183. We don't do so during normalization. This is a breaking change, but the affected code is very much unsound: https://github.com/rust-lang/trait-system-refactor-initiative/issues/253.

https://github.com/rust-lang/trait-system-refactor-initiative/issues/27

TODO: link to source

## Leak check?

make sure https://github.com/rust-lang/rust/pull/119820 is in the dev-guide :thinking:

Behavior between the two olvers is the same since https://github.com/rust-lang/rust/pull/146725, not quite https://rust-lang.zulipchat.com/#narrow/channel/364551-t-types.2Ftrait-system-refactor/topic/HRTB.20oddity/with/623184908

## Region uniquification

damn, wtf is that shit
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/30

## Overlapping impl candidates are blocking :>

https://github.com/rust-lang/trait-system-refactor-initiative/issues/35

## GAT where-bounds vs old solver?

https://github.com/rust-lang/trait-system-refactor-initiative/issues/44

## Avoid assembling impls shadowed by where-bounds

https://github.com/rust-lang/trait-system-refactor-initiative/issues/226

## looky look closures with non-identity args

https://github.com/rust-lang/trait-system-refactor-initiative/issues/243
