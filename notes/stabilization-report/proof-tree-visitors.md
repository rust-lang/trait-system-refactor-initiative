# Proof tree visitors

They exist because `FulfillmentCtxt` no longer contains nested obligations. Here are all the `ProofTreeVisitors` and some notes about them.

## `fn obligations_for_self_ty` and `fn pending_obligations_potentially_referencing_float_infer`

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/fn_ctxt/inspect_obligations.rs#L41

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/fn_ctxt/inspect_obligations.rs#L190

## `CoerceUnsized`

`coerce_unsized` https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/coercion.rs#L687-L700

coerce https://github.com/rust-lang/trait-system-refactor-initiative/issues/261