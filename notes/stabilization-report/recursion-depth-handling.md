# Recursion depth handling

## Non-fatal overflow

Encountering the recursion-limit is no longer fatal with the new trait solver. This allows us to remove some hacks, e.g. in [`ProbeContext::consider_probe`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/method/probe.rs#L2127-L2147) or [when checking goals for diagnostics](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/error_reporting/traits/ambiguity.rs#L88-L94). It also causes a bunch of problems.

As crates can successfully compile even if they hit the recursion limit, increasing the limit can worsen their compile-time performance. This affects `typenum` whose performance gets 2x worse when doubling the recursion depth.

This is partialy necessary due to the removal of [`fn match_fresh_trait_preds`](https://github.com/rust-lang/rust/blob/aea4dd4b0377fb5881542815dc3c2352394e8514/compiler/rustc_trait_selection/src/traits/select/mod.rs#L1213-L1226) https://github.com/rust-lang/trait-system-refactor-initiative/issues/56. We've removed this as it made the global cache observable, which is incorrect wrt incremental compilation.

Hitting overflow during `fulfill` is fatal, so are overflow errors in `query_normalize`. This is mainly as there's very little use in allowing that, it matches the old solver, and supporting non-fatal overflow here is challenging.

This means proving things slightly differently between HIR typeck and MIR borrowck can result in ICE. This is subtle and might end up being annoying to handle: https://github.com/rust-lang/trait-system-refactor-initiative/issues/238. Alternatively, we could change overflow during MIR borrowck to be fatal. This would not be breaking.

TODO: https://github.com/rust-lang/trait-system-refactor-initiative/issues/258 :<

### Discarding nested constraints on overflow

To avoid hangs, we drop nested constraints. This causes problems https://github.com/rust-lang/trait-system-refactor-initiative/issues/274

### Dividing the available depth when encountering overflow

https://github.com/rust-lang/trait-system-refactor-initiative/issues/276

### Long term plan

https://github.com/rust-lang/trait-system-refactor-initiative/issues/278

## Properly tracking the required `recursion_depth`

https://github.com/rust-lang/rust/pull/159224

https://github.com/rust-lang/rust/pull/162275