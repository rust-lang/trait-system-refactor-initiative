# Breaking Changes and Known Bugs

This is an overview of the currently known issues and breakage with the new solver and is ordered by impact.

This is currently based off [this spreadsheet](https://docs.google.com/spreadsheets/d/1Sq0kEbp31NiOzJ4iFlVGEtVAZH2B7Ya-uez_U1ffZ3g/edit?gid=1419391881#gid=1419391881), which is based off our most recent crater run. We will continue to look into the remaining failures and might fix some of them going forward, however, I think the breakage is minor enough that regardless of the remaining issues, we should go ahead with the staiblization.

The crater run had 114 failures and 310 spurious results. Of these 301 pass when testing locally and most are caused by a few root regressions. There are less than 150 affected crates found via crater. This is partially as a lot of the breakage has already been fixed. We're still looking into some of the results.

## Incompletely relating higher-ranked aliases

Intended inference breakage. See [the alias doc](./aliases-and-type-relations.md#on-demand-normalization) and https://github.com/rust-lang/trait-system-refactor-initiative/issues/168 for more info.

This causes by far the most breakage. It affected `bevy_ecs`, `minijinja`, and more. In the crater run this is down to 102 affected crates. In our crater run back in August this was 225 crates.

## Properly tracking required `recursion_depth` FCW

See [the overflow handling doc](./overflow-handling.md#properly-tracking-the-required-recursion_depth) and https://github.com/rust-lang/rust/issues/159228. We don't know its exact impact, it is the likely the change affecting the most users. This is only a future compat warning.

Looking at the change from https://github.com/rust-lang/rust/pull/133502#issuecomment-3358623722 to https://github.com/rust-lang/rust/pull/133502#issuecomment-3367902058, it likely affected more than 1000 crates, even if https://github.com/rust-lang/rust/pull/162275 has since lowered the impact. 

## Unconstrained assoc type of not-yet defined opaque type

Inference breakage we may fix after stabilization. See [the opaque types doc](./opaque-types.md#pseudo-rigid-inference-variables) and https://github.com/rust-lang/trait-system-refactor-initiative/issues/248. This broke 3 crates in a previous crater run, but these all been updated.

## Inductive cycles in fulfill resulted in errors

See [the cycle handling doc](./canonicalization-cycle-handling-and-caching.md#breaking-change) and https://github.com/rust-lang/trait-system-refactor-initiative/issues/224. This causes one root breakage and 3 total regressions. In the latest crater run there's still one affected project.

## Local projection cache strenghtening inference

Intended inference breakage. See [the caching doc](./canonicalization-cycle-handling-and-caching.md#inferctxt-local-caches-strengthen-inference) and https://github.com/rust-lang/trait-system-refactor-initiative/issues/215 for more info. This is known to break at least 2 different crates.

## Higher-ranked `TypeOutlives` bounds OOM

See There's also another memory use blowup affecting one crate on crater, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/308. Affects one crate. Will be fixed by the region constraints rework long-term, we may do something before then.

## Normalize via builtin impl in case of overlap

Intended breakage, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/101. There is one known regression.

## Other known bugs and open issues

See https://hackmd.io/HVoK3ysqSeqW_1s-KIzy7w for a list of all bugs and issues found on GitHub. We will continue fixing these going forward. 