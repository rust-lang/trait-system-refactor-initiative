# discard impls via env

If there is a `T: Trait` where-bound, then `<T as Trait>::Assoc` is well-formed but cannot be normalized. We say it is a *rigid* alias.

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

Without any explicit modifications, `NormalizesTo(<T as Trait>::Assoc)` will only see a single `Impl` candidate and no `ParamEnv` candidates, causing it to succeed. This results in multiple issues.

TODO: hiding impls sucks, not-hiding impls also sucks :3 figure out what to do