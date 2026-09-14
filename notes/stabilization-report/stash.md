# Stash

A temporary stash of things worthy of documentation found while working towards the stabilization.

## `if cx.next_trait_solver()` triage

closure signature infer https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/closure.rs#L390

`coerce_unsized` https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/coercion.rs#L687-L700

obligations_for_self_ty https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/fn_ctxt/inspect_obligations.rs#L41 https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/fn_ctxt/inspect_obligations.rs#L152

region dependent goals etc https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_analysis/src/check/check.rs#L2321

add_item_bounds_for_hidden_type https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/opaque_types/mod.rs#L297 is weird, what's going on there

evaluate not erroring for all goals which are known to error https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/mod.rs#L230

## `FIXME(-Znext-solver)` triage

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/region_infer/opaque_types/mod.rs#L384 we should yeet this as part of the opaque types FCP

## Entirely different type relations

`NextSolverRelate` vs `TypeRelating` :thinking: https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/at.rs#L145-L165

what exactly are the differences here?

`generalize` never tries to generalize non-rigid aliases https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/relate/generalize.rs#L163-L166

impact on https://github.com/rust-lang/trait-system-refactor-initiative/issues/8

non-rigid aliases can always be generalized to an infer var, so we always do so. Old solver does not know whether aliases are rigid, so it only does so when encountering an occurs check failure https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/relate/generalize.rs#L409

This is problematic for non-hr aliases in hr aliases https://github.com/rust-lang/trait-system-refactor-initiative/issues/110 incompleteness jank

handling of aliases with escaping bound vars is still scuffed https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/relate/generalize.rs#L554

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

## Caching the unconstrained inference variables of normalization

Minor breakage and jank https://github.com/rust-lang/trait-system-refactor-initiative/issues/215

https://github.com/rust-lang/trait-system-refactor-initiative/issues/275

## Avoid assembling impls shadowed by where-bounds

https://github.com/rust-lang/trait-system-refactor-initiative/issues/226

## looky look closures with non-identity args

https://github.com/rust-lang/trait-system-refactor-initiative/issues/243

## trait solving changes can impact runtime behavior

https://github.com/rust-lang/trait-system-refactor-initiative/issues/298

