# Candidate assembly

## Candidate preference

We now merge where-clauses by checking the constraits in their query response instead of a syntactic check. TODO: does this result in behavior differences. TODO: YES no constraints + MAYBE sus https://rust-lang.zulipchat.com/#narrow/channel/144729-t-types/topic/resolving.20equal.20regions/near/623504310

The candidate preference rules for `Trait` goals are the same as with the old solver since the FCP back in https://github.com/rust-lang/rust/pull/132325.

This is not the case for `Projection` goals. The old solver prefers builtin trait object candidates over user-written impls while the new solver does not, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/101.

We also still intentionally prefer builtin trait-object candidates over impls to avoid breakage: https://github.com/rust-lang/trait-system-refactor-initiative/issues/183. We don't do so during normalization. This is a breaking change, but the affected code is very much unsound: https://github.com/rust-lang/trait-system-refactor-initiative/issues/253.

https://github.com/rust-lang/trait-system-refactor-initiative/issues/27

## Avoid assembling impls shadowed by where-bounds

https://github.com/rust-lang/trait-system-refactor-initiative/issues/226

## 