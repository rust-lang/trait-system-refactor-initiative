# Overflow handling

Encountering the recursion-limit is no longer fatal with the new trait solver. For this to not result in exponential blowup and hangs, we've added some heuristics and hacks. This is something we can and should continue to improve post-stabilization. There are other limits in the type system which still result in fatal errors. 

## Non-fatal overflow

This allows us to remove some hacks, e.g. in [`ProbeContext::consider_probe`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/method/probe.rs#L2127-L2147) or [when checking goals for diagnostics](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/error_reporting/traits/ambiguity.rs#L88-L94). It also causes a bunch of problems.

As crates can successfully compile even if they hit the recursion limit, increasing the limit can worsen their compile time performance. This affects `typenum` whose performance gets 2x worse when doubling the recursion depth.

This is partialy necessary due to the removal of [`fn match_fresh_trait_preds`](https://github.com/rust-lang/rust/blob/aea4dd4b0377fb5881542815dc3c2352394e8514/compiler/rustc_trait_selection/src/traits/select/mod.rs#L1213-L1226) https://github.com/rust-lang/trait-system-refactor-initiative/issues/56. We've removed this as it made the global cache observable, which is incorrect wrt incremental compilation.

Hitting overflow during `fulfill` is fatal, and so are overflow errors in `query_normalize`. This is mainly as there's very little use in allowing that, it matches the old solver, and supporting non-fatal overflow here is challenging.

This means proving things slightly differently between HIR typeck and MIR borrowck can result in ICE. This is subtle and might end up being annoying to handle: https://github.com/rust-lang/trait-system-refactor-initiative/issues/238. Alternatively, we could change overflow during MIR borrowck to be fatal. This would not be breaking.

This also results in subtle new invariants of the type system. Whether we hit the overflow limit can differ between crates in the dependency graph, which could theoretically result in unsoudness: https://github.com/rust-lang/trait-system-refactor-initiative/issues/258
- difference in layout or `fn codegen_select_candidate` depending on whether a goal overflows
- treating overflow results as a proof of something not being possible, e.g. in `fn impossible_predicates`

I think we're currently fine here. It is an annoying invariant to keep in mind however.

### Discarding nested constraints on overflow

To avoid hangs, we drop nested constraints if a goal encountered overflow: [source](https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_next_trait_solver/src/solve/eval_ctxt/mod.rs#L1586-L1606). This is necessary as it's otherwise very easy to get exponentially large types which results in hangs and out of memory errors. Discarding these constraints does result in some issues, e.g. https://github.com/rust-lang/trait-system-refactor-initiative/issues/274. 

### Dividing the available depth when encountering overflow

Another source of exponential blowup is overflow where there are multiple candidates or overflowing goals per step. This way it's easy to get an exponential amount of goals. We avoid this by dividing the remaining available depth for nested goals by 4 once at least one nested goal hit the overflow limit: [source](https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_type_ir/src/search_graph/mod.rs#L288-L319).

This is necessary for `typenum`, see https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_type_ir/src/search_graph/mod.rs#L288-L319. However, it can unfortunately also result in breakage, even if there's currently no known affected project https://github.com/rust-lang/trait-system-refactor-initiative/issues/276.

## Properly tracking the required `recursion_depth`

The old solver does not store the required depth for a goal in its cache. There are a bunch of crates which rely on that.

To avoid breakage, we're now rerunning overflowing goals with twice the available depth and emit a FCW if that succeeds, see https://github.com/rust-lang/rust/issues/159228.

To reduce the impact of tracking the recursion depth correctly, we're also not increasing the required depth when proving auto traits for opaque types and coroutine witnesses https://github.com/rust-lang/rust/issues/159228.

## Other uses of arbitrary limits in the type system

While reaching the recursion limit inside of the trait solver is no longer fatal, there are other places which could overflow and therefore have an arbitrary limit in the type system. These are all a lot less important, but let's quickly go through some relevant ones.

[Rerunning cycle heads inside of the trait solver](./canonicalization-cycle-handling-and-caching.md#rerunning-canonical-goals-until-reaching-a-fixpoint) has an arbitrary limit of 8. In general, we tend to each reach a fixpoint after 1 iteration or fail with overflow. I am not aware of any non-artificial examples which even get close to this limit. Hitting the limit results in a non-fatal trait solver overflow. This arbitrary limit does result in theoretical breakage https://github.com/rust-lang/trait-system-refactor-initiative/issues/118.

[Overflow in the `FulfillmentContext`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_trait_selection/src/solve/fulfill.rs#L203-L215) pretty much only happens due to compiler bugs and remains fatal.

Each [`ProofTreeVisitor`](./proof-tree-visitors.md) has an artificially low limit to avoid performance issues: [source](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_trait_selection/src/solve/inspect/analyse.rs#L382-L384). Hitting these limits simply stops the visitors from recursing further. This is somewhat hacky, but should be good enough for now.

## Long term plan

My long term goal is to remove the reliance on non-fatal overflow again. Ideally we'd have some way to detect diverging paths in the trait solver and abort them because of that.

This is hard to do soundly if we have a global cache as it very easily makes goals depend on the current stack. We must also avoid breaking code which would otherwise compile.

The current setup works well enough, even if it is very much not ideal. I opened https://github.com/rust-lang/trait-system-refactor-initiative/issues/278 to track this.


