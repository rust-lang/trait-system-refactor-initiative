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

## Non-fatal overflow

Encountering the recursion-limit is no longer fatal with the new trait solver. This allows us to remove some hacks, e.g. in [`ProbeContext::consider_probe`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/method/probe.rs#L2127-L2147) or [when checking goals for diagnostics](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/error_reporting/traits/ambiguity.rs#L88-L94). It also causes a bunch of problems.

## Entirely different type relations

`NextSolverRelate` vs `TypeRelating` :thinking: https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/at.rs#L145-L165

what exactly are the differences here?

`generalize` never tries to generalize non-rigid aliases https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/relate/generalize.rs#L163-L166

impact on https://github.com/rust-lang/trait-system-refactor-initiative/issues/8

non-rigid aliases can always be generalized to an infer var, so we always do so. Old solver does not know whether aliases are rigid, so it only does so when encountering an occurs check failure https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/relate/generalize.rs#L409

handling of aliases with escaping bound vars is still scuffed https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/relate/generalize.rs#L554

## New `FulfillmentContext`

## Reimplementing `select`

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/solve/select.rs#L20

## Reimplementing rustdoc auto-trait impl generation

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/auto_trait.rs#L84-L91

## Const generics

uwu https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/mod.rs#L460

## When do we use `evaluate_obligation`

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/query/evaluate_obligation.rs#L86

## Candidate preference

We now merge where-clauses by checking the constraits in their query response instead of a syntactic check.

## Query normalize is gone, is that useful?

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/query/normalize.rs#L79