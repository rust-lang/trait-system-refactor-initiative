# Stabilize `-Znext-solver=globally`

This is the main stabilization proposal for the next-generation trait solver. Large and impactful changes are handled in separate FCPs.

## Performance impact

## Minor changes to type inference

### Eagerly evaluating nested goals

We removed the split between evaluation and fulfillment. This impacts type inference in two minor ways.
- selection now runs nested goals until reaching a fixpoint, slightly strengthening inference https://github.com/rust-lang/trait-system-refactor-initiative/issues/102
- selection currently does not try to prove nested goals if there's only one candidate, which changes the order in which we evaluate goals and is theoretically breaking https://github.com/rust-lang/trait-system-refactor-initiative/issues/97

## Fun Facts

These are not changes from the old solver, but instead interesting observations made while working towards its stabilization.

We can't actually require `T: Trait` to hold for rigid `<T as Trait>::Assoc` alias due to missing implied bounds https://github.com/rust-lang/trait-system-refactor-initiative/issues/177. Proving `T: Trait` has stronger requirements than normalizing associated types.

We also have to be careful to avoid query cycles. This means we won't even attempt to use impls for normalziation if they are shadowed by a where-clause as fetching `type_of` the associated item can cause query cycles, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/173.

Similarly, we have to first attempt to prove all nested where-clauses of an impl before fetching its associated items, as not doing so also results in query cycles when calling `type_of`, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/185.

---

This FCP is about the stabilization as a whole. It relies on the a list of self-contained FCPs and changes.

Given these changes this FCP exists to evaluate the following:
- 


## References

https://hackmd.io/7VnJO-qnSleVaMYZKyOGkA