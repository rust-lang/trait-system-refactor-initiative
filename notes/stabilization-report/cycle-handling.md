# Next-generation trait solver cycle handling

The next-generation trait solver handles cycles differently than the old solver.

- cycle kind is now "MAX of step kind" instead of "MIN"
    - necessary to handle https://github.com/rust-lang/trait-system-refactor-initiative/issues/10
- explicitly 3 different cycle kinds, why NoSolution, why ambig, why yes

## Allows more code to compile

https://github.com/rust-lang/trait-system-refactor-initiative/issues/114#issuecomment-3073743088

```rust
trait Trait {
    type Assoc;
}

fn foo<T: Trait<Assoc = <T as Trait>::Assoc>>(_: T::Assoc) {}
```

## Breaking change

In the old solver non-productive cycles are always ambiguous in `evaluate`. Tthe old solver only uses `evaluate` to select candidates and then processes these candidates in `fulfill`. This means `fulfill` also needs to handle cycles. We currently treating cycles in fulfill as an error, which can impact method selection. https://github.com/rust-lang/trait-system-refactor-initiative/issues/224

## TODO

we normalize inside of goals if there are hr aliases, does this mess us cycle detection?

vibe: negative reasoning and solver cycles
https://github.com/rust-lang/trait-system-refactor-initiative/issues/122