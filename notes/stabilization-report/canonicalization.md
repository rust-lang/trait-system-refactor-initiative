# Canonicalization

## Old-style canonicalization

We still use the old style canonicalization in some places, especially if the code is shared by both trait solvers.

We should remove one of them soon after stabilization https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/canonical/query_response.rs#L92-L108
We still 