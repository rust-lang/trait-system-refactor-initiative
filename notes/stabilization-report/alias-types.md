# Next-generation trait solver alias types handling

The next-generation trait solver explicitly encodes the concept of whether an alias is rigid in the representation of types. See https://github.com/rust-lang/rust/pull/156742. This differs from the old trait solver.

We also add support for on-demand normalization of aliases during type relations and in the trait solver itself. When encountering an alias not marked as rigid in type relations, we emit a `Projection` goal to normalize it at this point. This causes type relations to now emit nested `Projection` obligations. When doing a probe, we generally want to eagerly try to prove these and we change some places in the compiler to do so, e.g. [`Coerce::unify_raw`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/coercion.rs#L181-L192) and [`FnCtxt::try_find_coercion_lub`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/coercion.rs#L1355-L1363).

At the end of HIR typeck, writeback now explicitly normalizes all non-rigid aliases. This fixes a bunch of bugs around unnormalized aliases in the MIR body or during MIR building. It also allows us to remove a bunch of redundant normalization calls e.g. in [`TypeChecker::ascribe_user_type_skip_wf`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/canonical.rs#L291), [`TypeChecker::equate_normalized_input_or_output`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/input_output.rs#L235-L247),[`TypeChecker::relate_type_and_user_type`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/mod.rs#L493-L504)

## `ParamEnv` normalization

still do it, rigid alias handling sus

`reveal_opaque_types_in_bounds` https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_middle/src/ty/mod.rs#L1215 TODO

what is `with_normalized` https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_middle/src/ty/mod.rs#L1209