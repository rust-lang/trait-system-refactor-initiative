# Next-generation trait solver opaque type handling

The new solver includes a near complete rewrite of the way we handle opaque types:
- we always normalize opaque types to their hidden type in the defining scope.
- we introduce the concept of *non-defining* - but revealing - uses in the defining scope.
- to support recursive function calls, we have a few type inference hacks for *not-yet defined* opaques in their defining scope.


## High level mental model

Opaque types are aliases to their underlying type, the same way an associated type is an alias to the type specified in the relevant impl. This alias is *rigid* outside of the defining scope, and always normalizes to the underlying type inside of the defining scope. There are some hacks which treats them as kind of rigid in HIR typeck for the sake of type inference and backwards compatability. We'll discuss these later on.

There are *defining* and *non-defining* uses of an opaque type, depending on whether the generic arguments of the opaque are generic parameters. A use during HIR typeck is defining if all type and const arguments are generic parameters, while a use during MIR borrowck is defining if all arguments are generic parameters, including regions. If the arguments to the opaque type are defining, but the hidden type is not fully inferred, this is also considered a non-defining use.

Whenever we encounter an opaque type in its defining scope we normalize it via a `Projection` goal. We never keep an opaque as rigid in its defining scope, allowing us to remove a bunch of hacks during MIR building and in the MIR itself, e.g.
- [`ProjectionElem::OpaqueCast`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_public/src/mir/body.rs#L905)
- [`RustcPatCtxt::reveal_opaque_ty`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_pattern_analysis/src/rustc.rs#L127)
- [`TypeChecker::relate_type_and_user_type`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/mod.rs#L467-L477)
- [`NllTypeRelating::relate_opaques`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/relate_tys.rs#L116)
- [`NllTypeRelating::tys`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/relate_tys.rs#L428-L436) the call to `super_combine_tys` is fallible

Looking up an opaque type in the `opaque_type_storage` is currently a structural lookup. The current state is an intermediate step towards effectively using higher-kinded inference variables to infer the hidden types of opaque types. See https://rust-lang.zulipchat.com/#narrow/channel/364551-t-types.2Ftrait-system-refactor/topic/opaque.20types.20high.20hopes/with/584631784.

TODO: will this make our long term goal worse? :<

## `TypingMode`

The behavior of the trait solver differs depending on the current `TypingMode`, which explicitly represents the different stages during compilation. Handling opaque types now relies on 2 additional `TypingMode`.

Opaque types are handled as follows, depending on the stage we're in:
- during `TypingMode::Coherence` normalizing opaque types is always ambiguous [source](https://github.com/rust-lang/rust/blob/aea4dd4b0377fb5881542815dc3c2352394e8514/compiler/rustc_next_trait_solver/src/solve/project_goals/opaque_types.rs#L28-L42)
- we use `TypingMode::Typeck` only during HIR typeck. This is used to infer the hidden type of the opaque modulo regions.
- anything after that uses `TypingMode::PostTypeckUntilBorrowck`. Here we already know the hidden type modulo regions and either ignore regions or infer them during borrowck.
- user-facing analysis after borrowck uses `TypingMode::PostBorrowck`. We now fully know the hidden type of opaques in the defining scope. 
- after analysis we normalize all opaque types by simply using `type_of` to get their underlying type

This allows us to remove a bunch of hacky handling in functions which are conceptually in the defining scope and which happen after HIR typeck, e.g. [`fn check_opaque_meets_bounds`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_analysis/src/check/check.rs#L415-L416). and [`fn check_coroutine_obligations`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/at.rs#L145-L165).

## Non-defining uses in the defining scope

We need to support uses of an opaque type whose arguments are not generic parameters. We still normalize these opaque types to their underlying type though. This is necessary as there are existing projects with recursive calls whose arguments are not generic parameters whose RPIT is treated as fully opaque with the old solver. As we now always normalize opaque types in their defining scopes, we need to support non-defining uses, e.g. in [wax-0.6](https://github.com/olson-sean-k/wax/blob/1afcda8318201afc04ebed06fea907d54fc1bf8c/src/token/mod.rs#L1058). We also frequently encounter recursive uses with local regions as arguments, e.g. in the [`gll`](https://github.com/rust-lang-nursery/gll/blob/3e82b327f5dff5a7ab2c7c20498b597a0d47b581/src/generate/rust.rs#L724-L727) crate. See https://github.com/rust-lang/types-team/issues/129 for more information.

## General implementation details

We store all uses of opaque types in their defining scope in the [`opaque_type_storage`](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_infer/src/infer/mod.rs#L170-L171). The storage is a map from the opaque type `AliasTy` to its hidden type. This means we rely on structural identity of the opaque type arguments for lookup. As the keys can reference inference variables, canonical queries can return duplicate entries. That's kind of ugly and these entries are currently stored in a separate list: [source](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_next_trait_solver/src/canonical/mod.rs#L539-L552). 

We provide the list of previous opaque type uses in the [`CanonicalInput`](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_type_ir/src/solve/mod.rs#L458). Opaque types are always normalized by using a `Projection` goal: [source](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_next_trait_solver/src/solve/project_goals/opaque_types.rs#L83-L113). This may register a new defining use. These get returned as part of the [`ExternalConstraints`](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_next_trait_solver/src/solve/eval_ctxt/mod.rs#L1684-L1694).

## HIR typeck

It's the responsibility of HIR typeck to figure out the hidden type of all opaque types in the defining scope. HIR typeck is shared by all nested bodies of a typeck root. At the end of HIR typeck, we require that there exists a defining for every opaque type defined by the current body: [source](https://github.com/rust-lang/rust/blob/e15ceccfc6209c15b6c4bc6352f6ec6bfe579eaa/compiler/rustc_hir_typeck/src/opaque_types.rs#L115-L215). Examples
- `opaque<T, U> = Vec<U>` defining use
- `opaque<T, T> = Vec<T>` non-defining use
- `opaque<T, u32> = Vec<T>` non-defining use
- `opaque<T, ?inf> = Vec<T>` non-defining use
- `opaque<T, U> = Vec<?inf>` non-defining use because of hidden type
- `opaque<T> = &'inf u32` defining use as HIR typeck ignores regions
- `opaque<'?inf, T>` defining use as HIR typeck ignores regions

If we found at least one defining use, we map the hidden type of that use to the defining scope of the opaque type, and then use that type to check all other uses of this opaque: [source](https://github.com/rust-lang/rust/blob/e15ceccfc6209c15b6c4bc6352f6ec6bfe579eaa/compiler/rustc_hir_typeck/src/opaque_types.rs#L139-L146). Given `opaque<T> = Vec<T>` and `opaque<?a> = ?b`, we'd use the defining use to check the non-defining use, constraining `?b` to `Vec<?a>`.

As checking non-defining uses guides type inference, we need to do so before type inference fallback, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/207. We do this via [`fn try_handle_opaque_type_uses_next`](
https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/lib.rs#L232) which does not error if there is no defining use yet.



`has_opaques_with_sub_unified_hidden_type` https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/mod.rs#L1132 and opaques_with_sub_unified_hidden_type

With the old solver we eagerly replaced opaque types in the return type with an inference variable via [`InferCtxt::replace_opaque_types_with_inference_vars`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/opaque_types/mod.rs#L26). We no longer do so with the new solver. This was necessary as the old solver sometimes incorrectly treated opaque types as rigid in their defining scope.

All of the inference guidance for not-yet defined opaque types checks whether an inference variable is sub-unified with the hidden type of an opaque, not the hidden type of an opaque type itself.

### Inference guidance for obligations involving not-yet defined opaque types

https://github.com/rust-lang/trait-system-refactor-initiative/issues/182

TODO: https://github.com/rust-lang/rust/pull/161414

Need to also go through blanket impls if the self-type is a not-yet inferred opaque type https://github.com/rust-lang/trait-system-refactor-initiative/issues/196. The blanket impl handling here is kinda scuffed, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/205 / https://github.com/rust-lang/trait-system-refactor-initiative/issues/229

### Method calls on non-yet defined opaque types

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

We reject candidates which would constrain an opaque or which would not hold if the opaque type were rigid, see [`ProbeContext::should_reject_candidate_due_to_opaque_treated_as_rigid`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/method/probe.rs#L2259-L2331). Some subtleties, e.g. https://github.com/rust-lang/trait-system-refactor-initiative/issues/285

To handle opaque types correctly when computing the `fn method_autoderef_steps` we also track the currently defined opaque types in the canonical input and [response](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/canonical/query_response.rs#L92-L108).

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/query/evaluate_obligation.rs#L23

### Function calls on not-yet defined opaque types

https://github.com/rust-lang/trait-system-refactor-initiative/issues/181

### Other places treating opaque types as rigid

## MIR borrowck

Fun stuff, TODO link to PR and a bit of explanation

https://github.com/rust-lang/trait-system-refactor-initiative/issues/264

## Lints and MIR building

## Miscellaneous changes and open issues

We now always require defining scopes to actually provide a value for the hidden type of an opaque type and not doing so now eagerly results in a hard error. This means we no longer have to provide a default value in [`fn type_of` for RPITs](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_analysis/src/collect/type_of/opaque.rs#L260-L269). This is a minor breakage as it causes the following to now error:
```rust
fn test() -> impl Sized {
    test()
}
```

We allow equating two opaques which are both in their defining scope as we just unify their hidden types. The old solver explicitly errored here https://github.com/rust-lang/trait-system-refactor-initiative/issues/29.

The fact that we can always normalize opaque types in their defining scope means that proving auto-trait bounds for opaque types in their defining scope no longer fails with ambiguity https://github.com/rust-lang/trait-system-refactor-initiative/issues/32

We previously didn't normalize opaque types when checking region constraints. Doing so allows more code to compile: https://github.com/rust-lang/trait-system-refactor-initiative/issues/112

Opaque types in dead code still getting defined in MIR borrowck, constraining regions via member constraints https://github.com/rust-lang/trait-system-refactor-initiative/issues/170

Applying member constraints can be incomplete. This means new non-defining uses can theoretically result in unnecessary region constraints https://github.com/rust-lang/trait-system-refactor-initiative/issues/227

There's one weird footgun for `Copy` and `FnMut` closures. We should lint there or sth https://github.com/rust-lang/trait-system-refactor-initiative/issues/230 

There are places which currently use `structurally_resolve_type` which break with the new solver and opaque types https://github.com/rust-lang/trait-system-refactor-initiative/issues/231

## The shiny future

https://github.com/rust-lang/trait-system-refactor-initiative/issues/271 / https://rust-lang.zulipchat.com/#narrow/channel/364551-t-types.2Ftrait-system-refactor/topic/opaque.20types.20high.20hopes/with/619467087