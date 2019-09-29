- Feature Name: `version-aware-inference`
- Start Date: 2019-09-29
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

End insta-stable trait `impl`s and reduce stable-to-stable regressions by choosing longest-stable trait `impl` when otherwise ambiguous.

# Motivation
[motivation]: #motivation

Today, `#[stable(…)]` or `#[unstable(…)]` attribute on `impl`s of a trait is no-op and all trait `impl`s are "insta-stable".
However, it often causes type inference regressions *please fill some examples*. Such regressions are not desirable for the Rust compiler as a backward compatibility guaranteed software but in practice, those regressions are treated as acceptable because there was no way to solve this problem.

Here, I propose a way to solve the long-standing issue above in a generic manner, without affecting the semantics of the language.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When type inference cannot determine due to ambiguity, the compiler tries to solve type without using unstable or newer `impl`s.

```rust
//! On std
#[stable(feature = "rust1", since = "1.0.0")]
impl From<&str> for String { ... }

// new impl added in 1.35.0
#[stable(feature = "from_ref_string", since = "1.35.0")]
impl From<&String> for String { ... }


//! User code written before 1.35.0
fn use_string(string: impl Into<String>) { ... }

fn pass_string(string: String) {
    use_string(string.as_ref());
    // ^ Previously: error[E0283]: type annotations required: cannot resolve `std::string::String: std::convert::AsRef<_>`
    // ^ Now: Continue to working as expected.
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The cost

- Let *cost* of each trait `impl` as stabilization version.
  - If `stable` or `unstable` attribute with `since` argument is present, the cost is the value of `since` argument.
  - If `unstable` attribute is present but no `since` version is specified, the cost is infinity.
  - If `stable` nor `unstable` attribute is present, the cost is 0.
- Then define *cost* of a solution (of a trait resolution) as the maximum of the costs of the used `impl`s.

The cost is represented as a following enum:

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
enum Cost {
    StableUnspecified,
    Stable(Version),
    Unstable(Version),
    UnstableUnspecified,
}
```

Note that the ordering is `derive`d.

## Solve for the minimum cost

We want to find a solution with minimal cost solution. If there are multiple solutions with the same minimum cost, then it is an error.

This is backward compatible because if there is only one solution then it is the minimum cost one anyway.

NOT SURE ABOUT THIS: Trait solving is something like a BFS on a graph and a path is a solution. Multiple paths == ambiguous. It should be able to extend this BFS to find the [minimax path](https://en.wikipedia.org/wiki/Widest_path_problem) using Dijkstra's algorithm on max-min semiring.

## Linting

*TODO: Write motivation for this lint.*

Given the following function:

```rust
fn squash(cost: Cost) -> Cost {
    match cost {
        StableUnspecified => StableUnspecified,
        Stable(_) => StableUnspecified,
        Unstable(_) => UnstableUnspecified,
        UnstableUnspecified => UnstableUnspecified,
    }
}
```

If a unique solution can be determined with complete stabilization version awareness but it is no longer unique when `squash` is applied to the costs, a warning is emitted.

Stable and unstable are still distinct and nightly features can be introduced without warnings.

Efficient implementation: first solve on `squash`ed costs. If the solution is not unique, then solve original costs. This procedure calls solver only once when no warnings are emitted.

## Stability attributes on methods

```rust
trait TraitA {
    #[stable(since = "1")]
    fn provided_method(&self) {}
}

trait TraitB {
    #[stable(since = "2")]
    fn provided_method(&self) {}
}

#[stable(since = "1")]
impl TraitA for () {}

#[stable(since = "1")]
impl TraitB for () {}

fn f() {
    // should resolve to TraitA
    ().provided_method();
}
```

*I think* it can be implemented in the same way as trait `impl`s, such that `cost` is associated with the particular method lookup. Also, apply to non-trait methods? Useful when a conflict between trait method vs. inherit method etc. but it is unlikely to occur in `std` anyway. It is still better if more general is simpler.

*TODO: when I convinced there is no problem, replace phrases "`impl`s" to a more general term.*

# Drawbacks
[drawbacks]: #drawbacks

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

# Prior art
[prior-art]: #prior-art

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

- Generalize stability attributes to user crates. Probably its "version" should specify a crate version, not rustc version.
