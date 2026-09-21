# Stabilize `-Znext-solver=globally`

This is the main stabilization proposal for the next-generation trait solver. Large and impactful changes are handled in separate FCPs.

## Performance impact

## Breaking changes

We don't have an exact number here. We've had the new solver enabled on nightly for a while and all reported breakage has been tracked in https://github.com/rust-lang/rust/issues/160895. We separately did a bunch of crater runs in https://github.com/rust-lang/rust/pull/133502; including intended breakage we're at less than 500 affected crates. TODO: in more detail

## Future work

The new solver is far from perfect. We're partially just maintaining the status quo, but also some of our changes are not great and should be improved long term.
- opaque type handling and relying on structural identity, higher-ranked inference variables
- overflow handling, relying on hitting the recursion limit being non-fatal, `NestedGoals`
- borrowck being per body instead of per typeck root
- `ParamEnv` normalization is still shit

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

---

This FCP is about the stabilization as a whole. It relies on the a list of self-contained FCPs and changes.

Given these changes this FCP exists to evaluate the following:
- 


## References

https://hackmd.io/7VnJO-qnSleVaMYZKyOGkA