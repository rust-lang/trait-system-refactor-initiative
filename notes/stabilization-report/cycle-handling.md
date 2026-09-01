# Next-generation trait solver cycle handling

The next-generation trait solver handles cycles differently than the old solver.

- cycle kind is now "MAX of step kind" instead of "MIN"
- explicitly 3 different cycle kinds, why NoSolution, why ambig, why yes
- breaking change, fulfillment cycles are currently an error