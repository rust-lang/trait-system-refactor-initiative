# Canonicalization

## Rerunning canonical goals until reaching a fixpoint

This causes some minor breakage
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/209

## Old-style canonicalization

We still use the old style canonicalization in some places, especially if the code is shared by both trait solvers. We should remove one of them soon after stabilization.

We should remove one of them soon after stabilization https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/canonical/query_response.rs#L92-L108
We still 

## Erasing universe information in query inputs

that's cool stuff, TODO
- better for perf
- does not matter
- means that in https://github.com/rust-lang/rust/issues/161404 we don't trigger https://github.com/rust-lang/rust/blob/aea4dd4b0377fb5881542815dc3c2352394e8514/compiler/rustc_infer/src/infer/relate/generalize.rs#L425