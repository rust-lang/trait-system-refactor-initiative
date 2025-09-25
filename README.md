# trait-system-refactor-initiative

The Rustc Trait System Refactor Initiative is implementing the next-generation
trait solver: `-Znext-solver`.

- [Tracking Issue](https://github.com/rust-lang/rust/issues/107374)
- [zulip](https://rust-lang.zulipchat.com/#channels/364551/t-types.2Ftrait-system-refactor/general)

When working on the trait solver use `RUSTC_LOG=rustc_type_ir::search_graph=debug,rustc_next_trait_solver::debug` to get useful proof trees for it.

