# Proof tree visitors

They exist because `FulfillmentCtxt` no longer contains nested obligations. In the old solver we used `evaluate` to `select` a single candidate and then added the nested goals for that candidate to the `FulfillmentCtxt` and proved them later. The new solver does not have a split between proving goals in `fulfill` and `evaluate` so proving a goal in the `FulfillmentCtxt` directly proves all nested goals insteead of returning them.

There are still some parts of the type system which care about the nested obligations for a given root goal. For this we use [`ProofTreeVisitors`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_trait_selection/src/solve/inspect/analyse.rs#L377). This is an overview of interesting `ProofTreeVisitors` and why they exist.

## [`fn obligations_for_self_ty`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/fn_ctxt/inspect_obligations.rs#L108) and [`fn pending_obligations_potentially_referencing_float_infer`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/fn_ctxt/inspect_obligations.rs#L190)

There are a bunch of places during HIR typeck which look at the list of currently pending obligations to guide type inference. For compatibility with the old solver we're using a proof tree visitor to also look at nested obligations. We do this in the following locations.

When looking for `FnX` bounds for the `Expectation` in [`fn deduce_closure_signature`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_hir_typeck/src/closure.rs#L290). We do this similarly for [`async`-blocks](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_hir_typeck/src/closure.rs#L921) and [`async`-closures](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_hir_typeck/src/closure.rs#L524).

We also need to look at nested obligations in [`fn type_var_is_sized`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_hir_typeck/src/fn_ctxt/_impl.rs#L757). This is used by [`fn coerce_unsized`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_hir_typeck/src/coercion.rs#L2209) to decide whether to add a coercion from `?inf` to some unsized type.

## [`CoerceUnsized`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_hir_typeck/src/coercion.rs#L2169)

We're using a `ProofTreeVisitor` instead of [the manual fulfillment loop](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_hir_typeck/src/coercion.rs#L718-L821) used by old solver.

There were two bugs of the old implementation which are fixed by the new approach:
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/238
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/241

At it's core, the issue was that MIR typeck would simply prove `T: Unsize<U>` while the old solver manually handled these goals, which resulted in minor mismatches between HIR and MIR typeck. Handling these is annoying, whereas the new approach results in the same root obligation in both HIR and MIR typeck.

The old solver did not use where-clauses for incomplete inference guidance for coercions while the new solver does. This requires unstable features and I don't care about that inference change in general: https://github.com/rust-lang/trait-system-refactor-initiative/issues/261.

## [`BestObligation`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_trait_selection/src/solve/fulfill/derive_errors.rs#L411) for trait errors

With the old solver, successfully selecting a candidate puts its nested goals into the `FulfillmentCtxt`. If that nested goal then fails, we emit an error for the nested goal instead of the root goal:
```rust
fn is_clone<T: Clone>() {}
fn foo<T>() {
    is_clone::<Vec<T>>();
    //~^ ERROR: the trait bound `T: Clone` is not satisfied
}
```
This error emitting behavior is an implementation detail and also has some undesirable edge-cases, where we consider candidates we really shouldn't, e.g. talking about `Interator` instead of `IntoIterator`. This is also why we've added the `#[diagnostic::do_not_recommend]` attribute.

## [`AmbiguityCausesVisitor`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_trait_selection/src/traits/coherence.rs#L728) for coherence errors

Used by coherence to improve error messages in case of overlap. Emitting good errors here required changes to the old trait solver: [source](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_trait_selection/src/traits/select/mod.rs#L368-L398). 

## [`select`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_trait_selection/src/solve/select.rs#L38)

We originally didn't really want to have the concept of selecting an impl to exist in the trait solver as that did not fit well with our more logical perspective on what trait solving means.

We've somewhat gone back from that again, see [candidate assembly](./candidate-assembly.md) document.

There are some subtle difference between `select` with the new solver, old solver `select`, and the behavior of the trait solver. The new solver select never returns a candidate in case of ambiguity, as that's not necessary given the way its used.

Trait solving and selection can differ in the way they handle candidate preference. Trait solving only merges user-written impl and the builtin trait object impl candidates if they have the same constraints, while selection needs to always prefer the builtin trait object impl. This means we cannot use these two interchangeably https://github.com/rust-lang/trait-system-refactor-initiative/issues/241

I feel like our current approach here is quite outdated by now and we could probably remove this `ProofTreeVisitor` and replace it with something simpler. I think for now the current setup is fine.