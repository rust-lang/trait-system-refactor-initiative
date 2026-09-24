# Performance optimizations

## `FulfillmentContext`

TODO:

## `GoalEvaluation::stalled_on`

TODO:

## `TypingMode::ErasedNonCoherence`

We implemented a performance optimization to cache goals between HIR typeck and other parts of the compiler if they don't depend on the current `TypingMode`. This does not impact behavior, but significantly improves crates like `wg-grammar`. See https://github.com/rust-lang/rust/pull/155443.

The core idea is that instead of proving a goal in the current `TypingMode`, we may first run it with `TypingMode::ErasedNotCoherence`. We then track whether we did anything that relies on the current `TypingMode` and if so, we rerun this goal while providing the actual `TypingMode` this time.

## Avoid assembling impls shadowed by where-bounds

https://github.com/rust-lang/trait-system-refactor-initiative/issues/226