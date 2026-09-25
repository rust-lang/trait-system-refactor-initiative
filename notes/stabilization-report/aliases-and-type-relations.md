# Next-generation trait solver: aliases and type relations

The next-generation trait solver changes the way we handle aliases. This impacts both normalization and the way we relate types. Changes to the way we handle opaque types are discussed in [a separate document](https://github.com/rust-lang/trait-system-refactor-initiative/blob/main/notes/stabilization-report/opaque-types.md).
- we now explicitly track whether an alias is rigid in its current scope
- we normalize aliases on demand where necessary, e.g. in type relations

## Rigid alias marker

The next-generation trait solver explicitly encodes the concept of whether an alias is rigid in the representation of types https://github.com/rust-lang/rust/pull/156742. We did not track this explicitly with the old trait solver. The underlying concept has existed just the same and has not changed, we simply did not encode it explicitly.

An alias is always rigid wrt a given `TypingEnv`, i.e. the combination of the current `ParamEnv` and `TypingMode`. This means that moving a type between different `TypingEnv`s needs to change all contained aliases to be non-rigid again.

Changing the `ParamEnv` mostly happens by instantiating an `EarlyBinder`. This requires us to mark aliases as non-rigid either when wrapping it with the `EarlyBinder` or when instantiating that binder. We currently do the first, with `EarlyBinder` never containing any rigid aliases. This is unnecessary when using `instantiate_identity` while staying in a compatible `TypingEnv`, e.g. [using types from HIR typeck during MIR building](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_mir_build/src/builder/mod.rs#L510-L516). We handle this by explicitly marking aliases as rigid here again.

Handling changes to the `TypingMode` is a bit more fragile and requires us to be careful. This has caused some bugs in our refactoring, e.g. https://github.com/rust-lang/rust/pull/160125. Note that this is already an issue with the currently stable normalization approach as it also had the implicit concept of an alias being rigid.

There are very few places where we use different `ParamEnv`s in the same context. These also need to manually handle aliases. The main example here is [`fn check_type_bounds`](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_hir_analysis/src/check/compare_impl_item.rs#L2544-L2560). This concrete code is broken in two ways, both of which don't matter enough for me to even bother with writing a test:
- while we normalize the GAT in obligations, we don't normalize its occurances in the `ParamEnv`. Occurances of the GAT in where-clauses therefore remain rigid.
- normalizing with additional where-clauses can mark aliases as rigid because we end up shadowing an impl with a where-clause. This means the obligations now have aliases which are incorrectly marked as rigid.

Making this concept explicit is necessary for "on-demand normalization" to avoid performance issues and to support the "`ParamEnv` normalization jank". We'd otherwise try to renormalize rigid aliases whenever we encounter them.

## On-demand normalization

We also add support for on-demand normalization of aliases during type relations and in the trait solver itself. When encountering an alias not marked as rigid in type relations, we emit a `Projection` goal to normalize it at this point. This causes type relations to now emit nested `Projection` obligations. When doing a probe, we generally want to eagerly try to prove these and we change some places in the compiler to do so, e.g. [`Coerce::unify_raw`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/coercion.rs#L181-L192) and [`FnCtxt::try_find_coercion_lub`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_hir_typeck/src/coercion.rs#L1355-L1363).

This causes by far the most breakage of the stabilization. See the description of https://github.com/rust-lang/trait-system-refactor-initiative/issues/168. Trying to FCW here is very challenging and this is very much intended breakage. We should just accept this. See [the main stabilization proposal](https://github.com/rust-lang/trait-system-refactor-initiative/blob/main/notes/stabilization-report/meta.md#breaking-changes) for a complete overview of the resulting breakage.

It also fixes a bunch of other minor issues when relating higher-ranked associated types, e.g. https://github.com/rust-lang/trait-system-refactor-initiative/issues/9.

## Type relations and generalization

On-demand normalization is the largest conceptual change and has a bunch of other fallout on the way our type relations work. We reimplemented type equality for the new solver in [`SolverRelating`](https://github.com/rust-lang/rust/blob/919170b08cc014c8e85709dd80efeed2bac74562/compiler/rustc_type_ir/src/relate/solver_relating.rs#L43) while the still using [`TypeRelating`](https://github.com/rust-lang/rust/blob/919170b08cc014c8e85709dd80efeed2bac74562/compiler/rustc_infer/src/infer/relate/type_relating.rs#L15) with the old one. When encountering a non-rigid alias in a type relation, we replace it with an inference variable in the type relation itself before recursing: [source](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_type_ir/src/relate/solver_relating.rs#L197-L214). By doing so, we're fixing most of the issues when relating higher-ranked aliases: https://github.com/rust-lang/trait-system-refactor-initiative/issues/9.

### Generalization

We never encounter non-rigid aliases after that when relating types. Notably this also means that `generalize` never has to handle the `?x = <? as Trait>::Assoc` case, simplifying its implementation: [source](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_infer/src/infer/relate/generalize.rs#L148-L193).

The fact that we now have an explicit `IsRigid` marker also allows generalization to always replace non-rigid aliases with inference variables, instead of only doing it if there is an occurs check failure: [source](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_infer/src/infer/relate/generalize.rs#L411-L412). Unfortunately, higher-ranked aliases are still kind of scuffed and we need to keep most of the old solver complexity here.

The way we handle higher-ranked aliases is still incomplete in some cases, even if it's significantly better than with the old solver. There are some remaining issues.

Generalization still keeps non-rigid higher-ranked aliases around, which can be incomplete. See https://github.com/rust-lang/rust/issues/161404 for an example. Fixing this properly likely relies on higher-kinded inference variables. We could alternatively defer type relations when entering a binder to avoid this incompleteness.

Because we now always normalize non-rigid aliases before relating them, even if the alias was higher-ranked, we must be careful to not generalize inference variables inside of them. That can otherwise result in unavoidable ambiguity errors: [source](https://github.com/rust-lang/rust/blob/32e1cf827d2a7880bb40efbe35e6391d9ba6225d/compiler/rustc_infer/src/infer/relate/generalize.rs#L535-L555).

### `fast_reject`

Having an explicit `IsRigid` marker also allows `fast_reject` to structurally relates rigid aliases, which slightly improves compile time performance: [source](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_type_ir/src/fast_reject.rs#L353-L365).

## `ParamEnv` normalization jank

The old trait solver does not support on-demand normalization and instead normalizes the `ParamEnv` in an unnormalized `ParamEnv`, incorrectly treating that one as if it were normalized. We originally tried to fix this issue with the new trait solver, but did not do so as it results in performance issues and interesting design questions, see [this zulip thread](https://rust-lang.zulipchat.com/#narrow/channel/364551-t-types.2Ftrait-system-refactor/topic/goodbye.20proper.20param_env.20normalization/with/594260464). Properly handling aliases during `ParamEnv` normalization resulted in the following issues:
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/89
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/176
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/216
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/219
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/246
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/265

While the implementation differs, the behavior is now the same as with the old solver. We explicitly mark aliases in the unnormalized `ParamEnv` used for normalization as rigid. The exact way this works is quite subtle. This has been implemented in https://github.com/rust-lang/rust/pull/158643.

The actual behavior on stable here is quite subtle. We do not mark constants as rigid as the old solver eagerly normalizes all constants in an empty environment: [source](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_trait_selection/src/traits/mod.rs#L439-L525). By keeping constants as non-rigid, the impl with the new solver is a bit simpler: [source](https://github.com/rust-lang/rust/blob/f45772eb69d6ed3cc23be40625411a75f9f32c9d/compiler/rustc_trait_selection/src/traits/mod.rs#L457-L492). This matters for currently unstable const generics features and is tracked in https://github.com/rust-lang/project-const-generics/issues/118. 

We also do not mark the normalized-to term of `Projection` clauses as rigid, as the old solver explicitly normalizes the [output of `project`](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_trait_selection/src/traits/project.rs#L645).

## Renormalize during writeback

This is likely not actually strictly necessary with the new solver and something we could have implemented with the old one as well. Due to explicitly marking aliases as rigid, this is more performant with the new solver however.

At the end of HIR typeck, writeback now explicitly normalizes all non-rigid aliases. This fixes a bunch of bugs around unnormalized aliases in the MIR body or during MIR building. It also allows us to remove a bunch of redundant normalization calls e.g. in [`TypeChecker::ascribe_user_type_skip_wf`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/canonical.rs#L291), [`TypeChecker::equate_normalized_input_or_output`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/input_output.rs#L235-L247),[`TypeChecker::relate_type_and_user_type`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_borrowck/src/type_check/mod.rs#L493-L504)

## Deep normalization implementation

We replaced both [`normalize`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/normalize.rs#L35) and [`query_normalize`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/query/normalize.rs#L79) with a single unified [`deeply_normalize`](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_trait_selection/src/solve/normalize.rs#L171) folder. It differs from the old solver normalization routines in a few ways, none of which are too impactful.

We treat all aliases the same way: normalizing them by emitting a `Projection` goal. This matches the old solver's handling of associated types, but this now also applies to all other alias kinds: [source](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_next_trait_solver/src/normalize.rs#L135). This means the normalization folder no longer needs to keep track of the recursion depth itself as the folder itself is guaranteed to not diverge, even as proving any single `Projection` goal may overflow.

The normalization folder also handles ambiguous normalization of aliases with escaping bound vars slightly differently. The old implementation can leak placeholders by constraining inference variables via ambiguous nested obligations later on. While theoretically observable, there is no known instance of this actually mattering: [old](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_trait_selection/src/traits/normalize.rs#L222-L233) [new](https://github.com/rust-lang/rust/blob/a4c14451a9c1e134bcdbc97e2a255739c20df6e8/compiler/rustc_next_trait_solver/src/normalize.rs#L55-L70). In general, bugs related to universe handling and type inference are rarely obserable, as the main region checking happens in MIR borrowck instead of during type inference itself.

## Historical notes

We went through a few implementation strategy for normalization. Here's a quick overview and some links.

We initially tried to exclusively rely on on-demand normalization as "lazy normalization". We never tried to entirely avoid eagerly normalization, as e.g. changing lints and MIR borrowck to normalize on-demand would have been quite involved, so we eagerly normalized in writeback early-on.

We entirely gave up on not eagerly normalizing, e.g. during HIR typeck, due to performance concerns and the fact a bunch of places actually do rely on us eagerly normalizing, e.g. for caching or the occurs check: https://github.com/rust-lang/rust/pull/155767.

The way we normalize has also changed over time. We initially had the concept of "one-step normalization", e.g. `<T as Trait>::Assoc` would first normalize to `<T as OtherTrait>::Assoc`, which can then be further normalized to `u32` or whatever. We originally considered alias-bound candidates for each normalizeable alias in this chain. That was unsound and also caused ICE during MIR borrowck:
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/6
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/77

We also initially related aliases via `AliasRelate` goals. These goals had 3 candidates:
- normalize lhs, equate with unnormalized rhs
- normalize rhs, equate with unnormalized lhs
- structurally relate aliases

This is horrible idea. It has a very bad perf impact, fails with ambiguity due to subtle reasons and is generally quite unworkable.

On-demand normalization without explicitly tracking whether an alias is rigid mean even will proper normalization, will still relied on `AliasRelate`, now implemented by fully normalizing both the lhs and rhs and then structurally relating them. We needed `AliasRelate` to make sure we only structurally relate rigid aliases and had no way of otherwise knowing whether aliases were rigid or not.

This meant we renormalized aliases every time we related them, which is quite bad for perf. That's why we introduced rigid alias markers in the end.