- Start Date: 2014-08-17
- RFC PR #: (leave this empty)
- Rust Issue #: https://github.com/rust-lang/rust/issues/16541

# Summary

A `Combine` trait is added with a single function `combine` which sugars to
the `++` operator. The `Add` implementation for `String` is replaced by one
for `Combine`.

# Motivation

As @huonw mentions in https://github.com/rust-lang/rust/issues/14482, mathematical
convention is that addition is commutative. The stdlib already follows mathematical
convention in some cases (for example, `Zero` requires `Add` and is expected to
be the additive identity; ditto for `One` and `Mul`).

It is an (often unstated) assumption in many algorithms that `+` is a commutative
operator. Violating this assumption in the stdlib forces programmers to memorize
that `+` means something different in rust than it does in common language, and
also risks encouraging abuse of operator overloading.

There is a postponed proposal regarding having unit tests for Traits which enforce
invariants; commutativity of `+` is an natural one and it would be bad if it was
unenforcable because standard library was violating it.

# Detailed design

Currently everything in the stdlib which implements `Add` implements `add` as a
commutative operator, except for strings. Therefore I propose:
- Introduce a `Combine` trait with a `combine` function that sugars to the `++`
operator. The precedence of `++` is between the bitwise operators and the comparison operators.
- Implement this on `String` for concatenation. This replaces `Add` for `String`.
- Implement this for every collection, using the same semantics as their `Extendable`
  implementations. (`Vec`s and `Bitv`s would be concatenated, sets and maps would
  be unioned, etc.)
- Implement this on `Path` as a synonym for `join`
- Add "must be commutative" to the documentation for `Add`.
- Add "must be associative" to the documentation for `Combine`.

The signature of `combine` is exactly the same as that for `add` and the other
binary operators:

````rust
pub trait Combine<RHS,Result> {
    /// The method for the `++` operator
    fn combine(&self, rhs: &RHS) -> Result;
}
````
and will be updated alongside the other binary-operation traits as the trait system
is revamped. (For example, adding `CombineAssign` for in-place `++=` or making
`Result` an associated item.)

For those interested in algebraic names, this makes `++` into a semigroup operator.
Users who want an abelian group can then use `Add+Zero+Neg` (or `Add+Zero+Sub`,
this ambiguity should probably be addressed in a later RFC related to fixing the
numeric traits once we have associated items); users who want an arbitrary group
can use `Mul+One+Div`; users who want a monoid can use `Combine+Default`, etc.

This way nobody is surprised by generic code which sometimes does the Wrong Thing,
but we avoid having a Haskell-like scenario where every category-theoretic object
is supported (with corresponding mental load). We just have a few binary operators
each with simple conventions.

# Drawbacks

Code which uses `+` to add strings will need to use `++` instead.

# Alternatives

Leave `+` as is.

# Unresolved questions

`Combine` should also be used for function compositions, at least for single-argument
functions `T->T`. How would this interact with our current/future coherence rules?

Where else should `Combine` be used?

