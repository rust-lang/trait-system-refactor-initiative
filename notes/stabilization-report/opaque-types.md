# Next-generation trait solver opaque type handling

## High level mental model

Opaque types are aliases to their underlying type, the same way an associated type is an alias to the type specified in the relevant impl. This alias is *rigid* outside of the defining scope, and always normalizes to the underlying type inside of the defining scope. There are some hacks which partially treat them as rigid in HIR typeck for the sake of type inference and backwards compatability. We'll discuss them later on.

There are *defining* and *non-defining* uses of an opaque type, depending on whether the generic arguments of the opaque are generic parameters. A use during HIR typeck is defining if all type and const arguments are generic parameters, while a use during MIR borrowck is defining if all arguments are generic parameters, including regions. 

Whenever we encounter an opaque type in its defining scope we normalize it via a `Projection` goal. We never keep an opaque as rigid in its defining scope, allowing us to remove a bunch of hacks during MIR building and in the MIR itself, e.g.
- [`ProjectionElem::OpaqueCast`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_public/src/mir/body.rs#L905)
- [`RustcPatCtxt::reveal_opaque_ty`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_pattern_analysis/src/rustc.rs#L127)
- [`TypeChecker::relate_type_and_user_type`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/mod.rs#L467-L477)
- [`NllTypeRelating::relate_opaques`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/relate_tys.rs#L116)
- [`NllTypeRelating::tys`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/relate_tys.rs#L428-L436) the call to `super_combine_tys` is fallible

## `TypingMode` and using the already inferred hidden type after HIR typeck

This allows us to remove a bunch of hacky handling in functions which are conceptually in the defining scope and which happen after HIR typeck, e.g. [`fn check_opaque_meets_bounds`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_analysis/src/check/check.rs#L415-L416). and [`fn check_coroutine_obligations`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/at.rs#L145-L165).

## HIR typeck

`try_handle_opaque_type_uses_next` and `handle_opaque_type_uses_next`

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/lib.rs#L232

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/lib.rs#L259

defining uses pre fallback and after fallback right before writeback

`has_opaques_with_sub_unified_hidden_type` https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/mod.rs#L1132 and opaques_with_sub_unified_hidden_type

With the old solver we eagerly replaced opaque types in the return type with an inference variable via [`InferCtxt::replace_opaque_types_with_inference_vars`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/opaque_types/mod.rs#L26). We no longer do so with the new solver. This was necessary as the old solver sometimes incorrectly treated opaque types as rigid in their defining scope.

### Inference guidance for obligations involving not-yet defined opaque types



### Method calls on not-yet defined opaque types

We want to treat opaque types as rigid when calling methods on them in their own defining scope:
```rust
fn foo(b: bool) -> impl IntoIterator<Item = u32> {
    if b {
        foo(false).into_iter().map(|x| x + 1).collect::<Vec<_>>()
    } else {
        vec![0; 10]
    }
}
```

We reject candidates which would constrain an opaque or which would not hold if the opaque type were rigid, see [`ProbeContext::should_reject_candidate_due_to_opaque_treated_as_rigid`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/method/probe.rs#L2259-L2331).

To handle opaque types correctly when computing the `fn method_autoderef_steps` we also track the currently defined opaque types in the canonical input and [response](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/canonical/query_response.rs#L92-L108).

### Other places where opaque types are partially treated as rigid

## MIR borrowck

## Lints and MIR building

## Miscellaneous changes and open issues

We now always require defining scopes to actually provide a value for the hidden type of an opaque type and not doing so now eagerly results in a hard error. This means we no longer have to provide a default value in [`fn type_of` for RPITs](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_analysis/src/collect/type_of/opaque.rs#L260-L269).