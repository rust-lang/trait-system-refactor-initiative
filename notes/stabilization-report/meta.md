# Stabilize `-Znext-solver=globally`

This is the main stabilization proposal for the next-generation trait solver. Large and impactful changes are handled in separate FCPs.

## Important high-level concepts

There are a lot of technical nuances and implementation choices here. Not all of them require the same amount of attention. There are a few changes to the way we think about the type system:
- we explicitly track whether aliases are rigid and normalize on-demand: [doc](./aliases-and-type-relations.md)
- we pretty much completely changed the way opaque types work: [doc](./opaque-types.md)
    - we now always normalize opaque types when in their defining scope
    - we support non-defining uses in the defining scope
    - we add the explicit concept of psuedo-rigid opaque types during HIR typeck
    - we split borrowck into two steps to support non-defining uses in nested bodies
- a solver cycle is now coinductive if at least one step is productive, previously all steps had to be: [doc](./canonicaliation-cache-and-cycle-handling.md#cycle-handling)
- reaching the overflow limit is now non-fatal: [doc](./recursion-depth-handling.md)
- we removed the split between selection (evaluate) and fulfillment, use proof tree visitors to get information about nested goals: [doc](./proof-tree-visitors.md)
- the trait solver canonicalizes at each step: [doc](./canonicaliation-cache-and-cycle-handling.md)

## Performance impact

TODO: up-to-date table by Jana :>

mostly neutral, sometimes slower, sometimes faster, a lot of space to optimize going forward.

## Breaking changes

We don't have an exact number here. We've had the new solver enabled on nightly for a while and all reported breakage has been tracked in https://github.com/rust-lang/rust/issues/160895. We separately did a bunch of crater runs in https://github.com/rust-lang/rust/pull/133502; including intended breakage we're at less than 500 affected crates. 

TODO: in more detail

## Future work

The new solver is far from perfect. We're partially just maintaining the status quo, but also some of our changes are not great and should be improved long term.
- opaque type handling and relying on structural identity, higher-ranked inference variables
- overflow handling, relying on hitting the recursion limit being non-fatal, `NestedGoals`
- borrowck being per body instead of per typeck root
- `ParamEnv` normalization is still shit

## MIR borrowck and dependence on region identity

MIR borrowck is intended to only reprove things already proven by HIR typeck. Because of this, we ICE if MIR borrowck fails to prove something. MIR borrowck starts out by replacing all free regions in the body with unique region variables. This means that while HIR typeck may prove `T: Trait<'a, 'a>`, MIR borrow instead proves `T: Trait<'a, 'b>`.

Unfortunately, there are a bunch of subtle ways in which the trait solver relies on regions being identical. These include:
- [accessing the `opaque_type_storage`](./opaque-types.md), which is a structural lookup in the current implementation
- [merging multiple applicable where-clause, alias-bound, or builtin candidates](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_next_trait_solver/src/solve/mod.rs#L314-L320)
- potentially [trait solver cycle fixpoint behavior](./canonicaliation-cycle-handling-and-caching.md#rerunning-canonical-goals-until-reaching-a-fixpoint)

There are also fast paths for structurally identical types, e.g. [in type relations](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/relate/solver_relating.rs#L148-L150).

This means that MIR typeck may actually fail to prove something proven in HIR typeck due to *region uniquification*. This has resulted in a bunch of ICE, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/30.

The way we handled this has changed a lot as we've encountered issues or changed the design of the trait solver, see https://github.com/rust-lang/rust/pull/145706 for the latest way we're dealing with this. This is somewhat hacky.

The old solver is less region dependent than the new one. It does not merge candidates, instead arbitrarily prefering earlier alias-bound and builtin trait object candidates: [source](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_trait_selection/src/traits/select/mod.rs#L1919-L1935). The way opaque types are handled does result in ICE here, but that is less due to region dependence and instead because it fully recomputes the hidden types of opaque types instead of using the type inferred by HIR typeck.

I think long-term we might be able to change the trait solver to not depend on whether regions are equal after all. Representing opaque types via higher-kinded inference variables causes their lookup to no longer require structural identity. Merging multiple candidates does not rely on region identity if we support OR-constraints. We want to do that regardless for marker traits. We've explicitly made sure that merging candidates is future compatible with this approach.

TODO: link to the code which actually requires certainty to be the same. this is blocking!

## The leak check and `VisibleForLeakCheck`

The behavior wrt higher-ranked region errors in the trait solver is mostly the same between the new and old solver. In the old solver we don't consider constraints from nested goals as [trait goals are evaluated in a `probe`](https://github.com/rust-lang/rust/blob/3670d2532bdf51abbe0b8fea22284d7ca340ffe3/compiler/rustc_trait_selection/src/traits/select/mod.rs#L1276-L1292) and [outlives obligations get entirely ignored](https://github.com/rust-lang/rust/blob/3670d2532bdf51abbe0b8fea22284d7ca340ffe3/compiler/rustc_trait_selection/src/traits/select/mod.rs#L747-L765) in evaluation.

As the new solver does not have different implementations for fulfill and evaluate, it always returns the region constraints of nested goals. This is unfortunately unsound due to a lack of assumptions on binders, and we therefore changed the trait solver to explicitly ignore region constraints from nested goals via a `VisibleForLeakCheck` marker https://github.com/rust-lang/rust/pull/155749.

The new solver nearly perfectly matches the old solver now. However, evaluate in the old solver does apply constraints from nested `Projection` obligations, as they can constrain otherwise unconstrained inference variables. This also allows `Projection` goals to otherwise influence its parent obligation by returning constraints from matching the impl header. This is one case where the the implementation of the new solver will actually weaken the leak check. I don't think anyone relied on this. See the test added in https://github.com/rust-lang/rust/pull/163271.

## rustdoc auto-trait impl generation

The way we compute the auto-trait implementations for rustdoc depends on old solver internals. For now we've implemented a far simpler and weaker alternative 
https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/auto_trait.rs#L187. This alternative is very limited however, see https://github.com/rust-lang/rust/issues/162274. We should improve this as we move forward.

## Minor changes to type inference

### Eagerly evaluating nested goals

We removed the split between evaluation and fulfillment. This impacts type inference in two minor ways.
- selection now runs nested goals until reaching a fixpoint, slightly strengthening inference https://github.com/rust-lang/trait-system-refactor-initiative/issues/102
- selection currently does not try to prove nested goals if there's only one candidate, which changes the order in which we evaluate goals and is theoretically breaking https://github.com/rust-lang/trait-system-refactor-initiative/issues/97

The removal of this difference also allows us to cleanup some existing hacks, notably in [`fn pred_known_to_hold_modulo_regions`](https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_trait_selection/src/traits/mod.rs#L230-L251).

## Fun Facts

These are not changes from the old solver, but instead interesting observations made while working on the new solver.

Minor changes to incompleteness or type inference in general can result in *runtime behavior changes*, most notably by incompletely rejecting some candidates during method selection, e.g. hgttps://github.com/rust-lang/trait-system-refactor-initiative/issues/298.

We can't actually require `T: Trait` to hold for rigid `<T as Trait>::Assoc` alias due to missing implied bounds https://github.com/rust-lang/trait-system-refactor-initiative/issues/177. Proving `T: Trait` has stronger requirements than normalizing associated types.

We also have to be careful to avoid query cycles. This means we won't even attempt to use impls for normalziation if they are shadowed by a where-clause as fetching `type_of` the associated item can cause query cycles, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/173.

Similarly, we have to first attempt to prove all nested where-clauses of an impl before fetching its associated items, as not doing so also results in query cycles when calling `type_of`, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/185.