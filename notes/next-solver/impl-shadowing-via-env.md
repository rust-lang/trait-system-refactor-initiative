# discard impls via env

If there is a `T: Trait` where-bound or alias-bound, then `<T as Trait>::Assoc` is well-formed but cannot be normalized. We say it is a *rigid* alias.

However, even if there is a where-bound, we may still be able to normalize the associated type using an impl.
```rust
trait Trait {
    type Assoc;
}

impl<T> Trait for T {
    type Assoc = u32;
}

fn definitely_uses_impl<T>(x: <T as Trait>::Assoc) -> u32 {
    x
}

fn shadowed_by_env<T: Trait>(x: <T as Trait>::Assoc) -> u32 {
    x //~ ERROR mismatched types
}
```

Without any explicit modifications, `NormalizesTo(<T as Trait>::Assoc)` will only see a single `Impl` candidate and no `ParamEnv` candidates, causing it to succeed. This results in multiple issues as we will explore below.

## Why shadowing

### The where-bound may differ from the impl

If the where-bound also has a projection bound, this bound may differ from the impl, e.g.
```rust
trait Trait {
    type Assoc;
}

impl<T> Trait for T {
    type Assoc = T;
}

fn foo<T: Trait<Assoc = Vec<U>>, U>(x: Vec<U>) -> T::Assoc {
    x
}
```
We want to prefer the `Projection` where-bound over the impl here.

### Using shadowed impls may result in overflow

```rust
trait Super {
    type SuperAssoc;
}

trait Trait: Super<SuperAssoc = Self::TraitAssoc> {
    type TraitAssoc;
}

impl<T, U> Trait for T
where
    T: Super<SuperAssoc = U>,
{
    type TraitAssoc = U;
}

fn overflow<T: Trait>() {
    let x: <T as Trait>::TraitAssoc;
}
```
- normalize `<T as Trait>::TraitAssoc`
    - no `ParamEnv` candidates, single impl candidate
        - normalize `<T as Super>::SuperAssoc`, via env
            - normalize `<T as Trait>::TraitAssoc` cycle

This cycle steps into into an impl where-bound which we then "step out of". We currently treat the cycle as unknown, but in the future such cycles should error as they are non-productive. At this point this will no longer be an issue.

### Using shadowed impls may add different region constraints

```rust
trait Trait<'a> {
    type Assoc;
}
impl Trait<'static> for u32 {
    type Assoc = u32;
}

fn bar<'a, T: Trait<'a>>() -> T::Assoc { todo!() }
fn foo<'a>()
where
    u32: Trait<'a>,
{
    bar::<'_, u32>();
}
```
Normalizing via the where-bound when calling `bar` constrains the region to `'a` while using the impl constrains it to `'static`.

We should generally prefer explicit where-bounds over impls as that's what the user wrote. This issue also exists for alias-bounds on GATs: https://github.com/rust-lang/rust/issues/133044#issuecomment-2500684786.

I feel like ideally we check whether the region constraints differ between the where-bound and the lower-priority candidate and apply the alias-bound/impl only if doing so adds no new constraints.

### Summary

Where-bounds and alias-bounds should cover impls and if they differ from the impl or are more general. Using the covered candidate may add undesirable constraints.

## Why shadowing sucks

### Adding bounds to the env breaks code

If where-bounds shadow impls, adding additional where-bounds causes code to go from WF to non-WF. Notably, this causes issues during MIR inlining and in `compare_impl_item`. 

This feels unavoidable regardless of whether we shadow by using the environment. Adding where-bounds can certainly change equality if the where-bound differs from the previously used impl.

### Breakage when checking whether where-bounds cover impls

```rust
trait Trait {
    type Assoc;
}
impl<T> Trait for T {
    type Assoc = T;
}

fn impls_trait<T: Trait>() {}
fn foo<T>() 
where
    <T as Trait>::Assoc: Trait,
{
    impls_trait::<T>();
}
```
The check whether we can use impls to normalize `<T as Trait>::Assoc` is cyclic:
- normalize `<T as Trait>::Assoc`, shadow via where-bound
    - `T eq <T as Trait>::Assoc`
        - normalize `<T as Trait>::Assoc`, cycle

If the cycle is non-productive, we use `NoSolution` as the initial
result and rerun after successfully normalizing to `T`. The next iteration then uses this provisional result and succeeds, shadowing the impl. This causes `<T as Trait>::Assoc` to be a rigid alias. Rerunning the cycle yet again now fails to equate `T` with the alias.

## Could we fix it?

### Always treating cycles during the shadowing check as non-productive

This would cause the above example compile and would avoid this breakage. However, it causes eager `ParamEnv` normalization to impact behavior. If the cyclic reasoning is non-productive during eager normalization, we'd normalize the where-bound to `T: Trait`. At this point, it would shadow any future normalizations of `<T as Trait>::Assoc`. It would also cause us to no longer shadow normalization of `<T::Assoc as Trait>::Assoc` as `<T as Trait>::Assoc` is no longer rigid.

```rust
trait Trait {
    type Assoc;
}
impl<T> Trait for T {
    type Assoc = T;
}

fn impls_trait<T: Trait<Assoc = T>>() {}
fn foo<T>() 
where
    <T as Trait>::Assoc: Trait,
{
    impls_trait::<T>(); // error in normalized env, ok outside :<
}
```

### Implement generalized "is_global" approach

As stated above, we only need shadowing in cases where the where-bound results in fewer constraints than using the impl or if `Projection` where-bounds differ from the impl.

We could try to treat more where-bounds as *global* to stop them from shadowing impls. This could be done via proving the where-bound via only impl-candidates succeeds when considering the resulting region constraints.

While this may work and seems like a desirable change long-term, it is a major change in itself:
- it allows a significant amount of new code to compile
- it may have a significant performance impact and could result in hangs
- its interaction with cycle handling is non-trivial
