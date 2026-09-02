# Next-generation trait solver cycle handling

The next-generation trait solver handles cycles differently than the old solver.

- cycle kind is now "MAX of step kind" instead of "MIN"
    - necessary to handle https://github.com/rust-lang/trait-system-refactor-initiative/issues/10
- explicitly 3 different cycle kinds, why NoSolution, why ambig, why yes

## Breaking change

In the old solver non-productive cycles are always ambiguous in `evaluate`. Tthe old solver only uses `evaluate` to select candidates and then processes these candidates in `fulfill`. This means `fulfill` also needs to handle cycles. We currently treating cycles in fulfill as an error, which can impact method selection. TODO breaking change