# Canonicalization, caching, and cycle handling

There are a bunch of technical details and minor changes related to canonicalization, caching, and the way trait solver cycles are handled. The most notable change is that cycles are now considered coinductive if they have at least one productive step, where a productive step is proving a where-clause of an impl. 

## Canonicalization

With the new trait solver we're now canonicalizing at every step in the trait solver. We've implemented a new canonicalization routine for this: [source](https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_next_trait_solver/src/canonical/mod.rs#L55). We still use the existing canonicalization routine in some places, more on that later.

The main differences between the canonicalization routines is as follows.
1. The old input canonicalization replaces regions with existential variables. With the new solver regions are replaced with placeholders instead. This is necessary to support [`VisibleForLeakCheck`](https://github.com/rust-lang/rust/pull/155749) and also fixes https://github.com/rust-lang/rust/issues/106569.
2. We canonicalize generic parameters in the input to placeholders. This means we no longer encounter `ty::Param` in the trait solver and may slightly improve perf by erasing the name of placeholders inside of the trait solver.
3. When canonicalizing, we track whether inference variables have been sub-unified. This is mainly necessary for [opaque types](./opaque-types.md). This means we can revert https://github.com/rust-lang/rust/pull/119989 after we've stabilized the new solver.
4. As mentioned in the [opaque types](./opaque-types.md) document, `CanonicalInput` contains a list of already defined opaque types.
5. We erase universe information in query inputs, more on that later.

We still use the old style canonicalization in some places, especially if the code is shared by both trait solvers. We should remove one of them soon after stabilization. There are a few subtle differences between them. We added some hacks to places which rely on old canonicalization to handle opaque types: https://github.com/rust-lang/rust/blob/70222712809cd5cc1718ed8995914a1cbacb6b92/compiler/rustc_infer/src/infer/canonical/query_response.rs#L92-L108.

### Erasing universe information in query inputs

All inputs get put into the root universe. The trait solver does not care about universes it cannot access and this improves caching. It does mean in https://github.com/rust-lang/rust/issues/161404 we don't trigger [an internal warning](https://github.com/rust-lang/rust/blob/aea4dd4b0377fb5881542815dc3c2352394e8514/compiler/rustc_infer/src/infer/relate/generalize.rs#L425).

Old canonicalization already does this for type and const inference variables for performance reasons: [source](https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_infer/src/infer/canonical/canonicalizer.rs#L355-L358).

We don't need to provide universe information to canonical queries as we are able to recover all universe information when instantiating query responses. Input placeholders get mapped back to their input universal variable: [source](https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_next_trait_solver/src/canonical/mod.rs#L266-L280).

We instantiate any new existential variables in the currently highest universe. They then get pulled down into their actual universe when equating them with the original `var_values`: [source](https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_next_trait_solver/src/canonical/mod.rs#L501-L504).


### Canonical `param_env` cache

Canonicalization has a cache to very quickly canonicalize the `param_env`
https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_next_trait_solver/src/canonical/canonicalizer.rs#L149. This is necessary as it has a huge impact on the compilation time of some crates, see https://github.com/rust-lang/rust/pull/141451#issuecomment-2959946611.

## Cycle handling

The next-generation trait solver handles cycles differently than the old solver. This change is necessary due to https://github.com/rust-lang/trait-system-refactor-initiative/issues/10. The old trait solver did not track cycle participants sufficiently. This change more closely matches my current intuition of what cycles are and how to deal with them. However, we're still far from fully figuring this out and there are some open questions we're going to ignore as part of this stabilization.

A cycle is now considered coinductive if at least one step is productive. In the old cycles were only coinductive if all goals involved in the cycle were coinductive. Importantly, whether a cycle is coinductive does not depend on the goals in the cycle, but the steps between goals; the reason why were proving nested goals: [source](https://github.com/rust-lang/rust/blob/28b5293debd90e0ad9b8ccb937c03e499cfc2170/compiler/rustc_next_trait_solver/src/solve/eval_ctxt/mod.rs#L430-L469). This does not affect too much code. See https://github.com/rust-lang/rust/blob/3670d2532bdf51abbe0b8fea22284d7ca340ffe3/tests/ui/traits/next-solver/cycles/coinduction/only-one-coinductive-step-needed.rs for an example of what's allowed now.

If a cycle is coinductive, its initial provisional value is `Certainty::Yes` with no constraints, otherwise we return overflow. In the medium term, we'll likely change cycles which are known to be unproductive to `NoSolution` instead, but that's not necessary for stabilization: https://github.com/rust-lang/rust/pull/163159.

### Breaking change

In the old solver non-productive cycles are always ambiguous in `evaluate`. It only uses `evaluate` to select candidates and then processes these candidates in `fulfill`. This means `fulfill` also needs to handle cycles. We currently treating cycles in fulfill as an error, which can impact method selection. https://github.com/rust-lang/trait-system-refactor-initiative/issues/224

### Weird jank

We still haven't fully figured out the way cycle handling should work.

On a conceptional level, normalizing aliases in a goal should happen *outside* of that goal. This happens accidentally if we eagerly normalize. However, we can't always eagerly normalize higher-ranked aliases, so these may get normalized inside of the goal. This may change the cycle kind and I can't tell whether this can cause any issues. I think that for now things are fine here.

I also haven't really figured out how negative reasoning interacts with our cycle handling, but think this shouldn't block stabilization, see https://github.com/rust-lang/trait-system-refactor-initiative/issues/122.

When proving goals via item bounds, this may hide cycles which we then never detect. This is a major bug, see https://github.com/rust-lang/rust/issues/135246 and https://github.com/rust-lang/rust/issues/150508.

### Rerunning canonical goals until reaching a fixpoint

Due to canonicalization, we detect cycles even if the relevant inference variables are different. Consider the following example
```rust
trait Foo {} // assume `Foo` is coinductive here
struct Wrapper<T>(T);

impl<T> Foo for Wrapper<Wrapper<T>>
where
    Wrapper<T>: Foo
{} 
```
Proving `Wrapper<?a>: Foo` instantiates `a` with `Wrapper<?b>` and then proves `Wrapper<?b>: Foo`. Due to canonicalization these two goals are the same. If `Foo` is a coinductive trait, then we return `Yes` from the cycle. However, this must not simply succeed, but must overflow instead.

The way this works is that initially when proving a goal, the `provisional_result` for a cycle depends on the `PathKind`: [source](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/search_graph/mod.rs#L1317-L1324). Once we finished proving a cycle head, we then check whether all provisional results used for this goal are equal to its result. If this is not the case, we set the `provisional_result` to the result of this iteration and try again until reaching a fixpoint: [source](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/search_graph/mod.rs#L1362-L1459).

Doing it this way instead of only detecting cycles if the goals are exactly equal results in minor breakage for goals which result in placeholder constraints: https://github.com/rust-lang/trait-system-refactor-initiative/issues/209. This will get fixed long-term by the region constraints rework https://github.com/rust-lang/goals/issues/621.

#### Avoiding exponential blowup

Rerunning cycle heads can result in exponential blowup for more involved cycles. This was a very large issue [while we tried to do proper `ParamEnv` normalization](./aliases-and-type-relations.md#paramenv-normalization-jank). As this is something we're not doing as part of this stabilization, these performance optimizations are still necessary, but significantly less so. 

We track [`HeadUsages`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/search_graph/mod.rs#L160-L179) and if a candidates ends up not impacting the result of a goal, we don't care whether this candidate depends on an outdated provisional result. We [drop irrelevant usages](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_next_trait_solver/src/solve/trait_goals.rs#L1618-L1643), and [then avoid rerunning because of it](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/search_graph/mod.rs#L1377-L1397).

We also never rerun if a goal is ambiguous with no constraints. We just return ambiguity in this case as an ambiguous provisional result really should not change the final result to not be ambiguous in the next iteration: [source](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/search_graph/mod.rs#L1399-L1415).

## Caching

The old solver has a lot of different caches at different levels, see [this potentially outdated list from when I started working on the new solver](https://hackmd.io/1OmN5Oj4SL-PzAYsJyktwA#the-current-solver). This is a mess and a bunch of these caches are subtly unsound, especially wrt incremental compilation.

Caching in the new trait solver is incredibly subtle, so I split the [`SearchGraph`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/search_graph/mod.rs#L1-L13) out into a separate component, with a fuzzer to help with its correctness: https://github.com/lcnr/search_graph_fuzz.

The new solver has [a single global cache](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_middle/src/ty/context.rs#L681-L682). This cache is used for all goals, so normalization does not use a separate cache. The global cache must not be obserable as that would be unsound wrt incremental.

This means the global cache keeps track of the required recursion limit: https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/search_graph/global_cache.rs#L88-L111.

However, the global cache by itself in insufficient to avoid exponential blowup and hangs. To deal with this we also have [the `provisional_cache`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/search_graph/mod.rs#L612-L616) which is only used while inside of a trait solver cycle. This cache is allowed to be observable, which is fine, because we only move the cycle root to the global cache and the provisional cache entries for all other cycle participants just get dropped.

There's a lot of nuance to the way the provisional cache works. The most involved part is likely [`rebase_provisional_cache_entries`](https://github.com/rust-lang/rust/blob/1a8fa555801329bd0e803d7384b5a21191c61f30/compiler/rustc_type_ir/src/search_graph/mod.rs#L997). I don't think going in-depth in this stabilization report is worth it.


### `InferCtxt`-local caches strengthen inference

The old solver has caches local to the current `InferCtxt`. For ambiguous normalization, this cache stores the returned unconstrained inference variable: [source](https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_infer/src/infer/mod.rs#L103).

Caching that `<?x as Trait>::Assoc` normalizes to a specific `?fresh_var` slightly strenghtened inference. Not doing so is a minor breaking change:
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/215
- https://github.com/rust-lang/trait-system-refactor-initiative/issues/275

The new solver also has some local caches, e.g. [in generalization](https://github.com/rust-lang/rust/blob/622fd6a3f80ff4398db552ed138243c845347298/compiler/rustc_infer/src/infer/relate/generalize.rs#L356). This means that we do sometimes normalize ambiguous aliases to the same inference variables.