# Next-generation trait solver alias types handling

## Rigid alias marker

The next-generation trait solver explicitly encodes the concept of whether an alias is rigid in the representation of types. See https://github.com/rust-lang/rust/pull/156742. This differs from the old trait solver.

An alias is always rigid wrt a given `TypingEnv`, i.e. the combination of the current `ParamEnv` and `TypingMode`. This means that moving a type between different `TypingEnv`s needs to change all contained aliases to be non-rigid again.

Changing the `ParamEnv` currently only happens by instantiating an `EarlyBinder`. This requires us to mark aliases as non-rigid either when wrapping it with the `EarlyBinder` or when instantiating that binder. We currently do the first, with `EarlyBinder` never containing any rigid aliases. This is unnecessary when using `instantiate_identity` while staying in a compatible `TypingEnv`, e.g. [using types from HIR typeck during MIR building](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_mir_build/src/builder/mod.rs#L510-L516). We handle this by explicitly marking aliases as rigid here again.

Handling changes to the `TypingMode` is a bit more fragile and requires us to be careful. This has caused some bugs in our refactoring. Note that this is already an issue with the currently stable normalization approach as it also had the concept of an alias being rigid, we simply did not track it explicitly.

Making this concept explicit is necessary for "on-demand normalization" to avoid performance issues and to support the "`ParamEnv` normalization jank". We'd otherwise try to renormalize rigid aliases whenever we encounter them.

## Deep normalization

https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/normalize.rs#L35 TODO

## On-demand normalization

We also add support for on-demand normalization of aliases during type relations and in the trait solver itself. When encountering an alias not marked as rigid in type relations, we emit a `Projection` goal to normalize it at this point. This causes type relations to now emit nested `Projection` obligations. When doing a probe, we generally want to eagerly try to prove these and we change some places in the compiler to do so, e.g. [`Coerce::unify_raw`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/coercion.rs#L181-L192) and [`FnCtxt::try_find_coercion_lub`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/coercion.rs#L1355-L1363).

By doing so, we're fixing most of the issues when relating higher-ranked aliases:
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/9

## Renormalize during writeback

At the end of HIR typeck, writeback now explicitly normalizes all non-rigid aliases. This fixes a bunch of bugs around unnormalized aliases in the MIR body or during MIR building. It also allows us to remove a bunch of redundant normalization calls e.g. in [`TypeChecker::ascribe_user_type_skip_wf`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/canonical.rs#L291), [`TypeChecker::equate_normalized_input_or_output`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/input_output.rs#L235-L247),[`TypeChecker::relate_type_and_user_type`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/mod.rs#L493-L504)

## `ParamEnv` normalization jank

The old trait solver does not support on-demand normalization and instead normalizes the `ParamEnv` in an unnormalized `ParamEnv`, incorrectly treating that one as if it were normalized. We originally tried to fix this issue with the new trait solver, but did not do so as it results in performance issues and interesting design questions, see [this zulip thread](https://rust-lang.zulipchat.com/#narrow/channel/364551-t-types.2Ftrait-system-refactor/topic/goodbye.20proper.20param_env.20normalization/with/594260464). Properly handling aliases during `ParamEnv` normalization resulted in the following issues:
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/89

We keep the behavior of the old solver but implement it differently. We explicitly mark aliases in the unnormalized `ParamEnv` used for normalization as rigid. The exact way this works is quite subtle, but its behavior should effectively match the old trait solver. This has been implemented in https://github.com/rust-lang/rust/pull/158643.

Interesting:
- we do not mark constants as rigid, only types. This ends up matching the existing stable behavior, it matters for currently unstable const generics features. That's tracked in https://github.com/rust-lang/project-const-generics/issues/118.

## TODO

normalization dev-guide chapter

`reveal_opaque_types_in_bounds` https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_middle/src/ty/mod.rs#L1215 TODO

what is `with_normalized` https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_middle/src/ty/mod.rs#L1209