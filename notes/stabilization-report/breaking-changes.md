# Breaking Changes and Known Bugs

This is an overview of the currently known issues and breakage with the new solver and is ordered by impact.

This is currently based off [this spreadsheet](https://docs.google.com/spreadsheets/d/1BOo-ZKkT8vBqCkcHnBJ8fkMpKnYIHESIkHC9p8Om-5g), which is based off our crater runs at the start of August. The numbers should have only improved and I don't think it's worth it to delay the stabilization FCP on doing another complete crater triage before then.

We have less than 500 true failures. In this crater run there are currently less than 200 regressions for which we have not figured out the root cause yet. Most of these are spurious regressions we couldn't easily build locally.

## Incompletely relating higher-ranked aliases

Intended inference breakage. See [the alias doc](./aliases-and-type-relations.md#on-demand-normalization) and https://github.com/rust-lang/trait-system-refactor-initiative/issues/168 for more info.

This causes by far the most breakage. It affected `bevy_ecs`, `minijinja`, and more. In the crater run this affected at least 225 crates. The biggest root causes were the following and have all done new point releases by now:
- minijinja-2.X.X with 132 regressions
- minijinja-1.X.X with 25 regressions
- bevy_ecs-0.15.X with 18 regressions
- tera-2.0.0 with 15 regressions

## Properly tracking required `recursion_depth` FCW

See [the overflow handling doc](./overflow-handling.md#properly-tracking-the-required-recursion_depth) and https://github.com/rust-lang/rust/issues/159228. We don't know its exact impact, it is the likely the change affecting the most users. This is only a future compat warning.

Looking at the change from https://github.com/rust-lang/rust/pull/133502#issuecomment-3358623722 to https://github.com/rust-lang/rust/pull/133502#issuecomment-3367902058, it likely affected more than 1000 crates, even if https://github.com/rust-lang/rust/pull/162275 has since lowered the impact. 

## Unconstrained assoc type of not-yet defined opaque type

Inference breakage we may fix after stabilization. See [the opaque types doc](./opaque-types.md#pseudo-rigid-inference-variables) and https://github.com/rust-lang/trait-system-refactor-initiative/issues/248. This is known to break at least 3 crates.

## Inductive cycles in fulfill resulted in errors

See [the cycle handling doc](./canonicalization-cycle-handling-and-caching.md#breaking-change) and https://github.com/rust-lang/trait-system-refactor-initiative/issues/224. This causes one root breakage and 3 total regressions.

## Local projection cache strenghtening inference

Intended inference breakage. See [the caching doc](./canonicalization-cycle-handling-and-caching.md#inferctxt-local-caches-strengthen-inference) and https://github.com/rust-lang/trait-system-refactor-initiative/issues/215 for more info. This is known to break at least 2 different crates.

## Normalize via builtin impl in case of overlap

Intended breakage, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/101. There is one known regression.

## Other known bugs and open issues

See https://hackmd.io/HVoK3ysqSeqW_1s-KIzy7w for a list of all bugs and issues found on GitHub. We will continue fixing these going forward. 