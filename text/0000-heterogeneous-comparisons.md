- Feature Name: heterogeneous_comparisons
- Start Date: 2017-06-06
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow to compare integer values of different signedness and size.

# Motivation
[motivation]: #motivation

Right now every comparison between different integral types requires a cast. These casts don't only clutter the code, but also encourage writing incorrect code.

The easiest way to compare signed and unsigned values is to cast unsigned value to signed type. It works most of the time, but it will silently ignore overflows.

Comparison between values of different integer types is always well-defined. There is only one correct result and only one way to get it. Allowing compiler to perform these comparisons will reduce both code clutter and count of hidden overflow/underflow errors users make.

# Detailed design
[design]: #detailed-design

`PartialEq` and `PartialOrd` should be implemented for all pairs of signed/unsigend 8/16/32/64/(128) bit integers and `isize`/`usize` variants.

Implementation for signed-singed and unsigned-unsigned pairs should promote values to larger type first, then perform comparison.

Example:

```
fn less_than(a: i8, b: i16) -> bool {
    (a as i16) < b
}
```

Implementation for signed-unsigned pairs where unsigned type is smaller than machine word size can promote both values to smallest signed type fully covering values of both argument types, then perform comparison.

Example:

```
fn less_than(a: i8, b: u16) -> bool {
    // i32 is the smallest signed type
    // which can contain any u16 value
    (a as i32) < (b as i32)
}
```

Implementation for signed-unsigned pairs where unsigned type is as big as machine word size or larger should first check if signed value less than zero. If not, then it should promote both values to unsigned type with the same size as larger argument type and perform comparison. For most platforms it should be possible to implement it without actual branching.

Example (for 32-bit system):

```
fn less_than(a: i64, b: u32) -> bool {
    (a < 0) || ((a as u64) < (b as u64))
}
```


Optionally `Ord` and `Eq` can be modified to allow `Rhs` type not equal to `Self`:

```
pub trait Eq<Rhs = Self>: PartialEq<Rhs> { }

pub trait Ord<Rhs = Self>: Eq<Rhs> + PartialOrd<Rhs> {
    fn cmp(&self, other: &Rhs) -> Ordering;
}
```

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

As of now I can't find anything about comparisons in "The Rust Programming Language". The most reasonable place seems to be in chapter "3.2 Data Types", in or just after "Numeric Operations" paragraph (for second edition).

We just need to mention that Rust allows comparison between different numeric types and that it returns mathematically correct result. Optionally, potential overhead can be mentioned, even though I'm not sure that's the right place to mention it.

Rust reference seems to describe language itself and doesn't go into details about any traits/operators implementation. I can't find any reasonable place to cover it there.

# Drawbacks
[drawbacks]: #drawbacks

* It might break some code relying on return type polymorphism. It won't be possible to infer type of the second argument from type of the first one for `Eq` and `Ord`.
* Correct signed-unsigned comparison requires one more operation than regular comparison. Proposed change hides this performance cost. If user doesn't care about correctness in his particular use case, then cast and comparison is faster.
* The rest of rust math prohibits mixing different types and requires explicit casts. Allowing heterogeneous comparisons (and only comparisons) makes rust math somewhat inconsistent.
* For some platforms it might be necessary or more efficient to use branching in signed-unsigned comparisons (arm thumb?). Using branching will turn comparison into a non-constant-time operation. Any value-dependant operation in cryptography code is a potential security risk because of timing attacks. On other hand, on some platforms even multiplication is not guaranteed to be a constant-time operation.

# Alternatives
[alternatives]: #alternatives

* Keep things as is.
* Add generic helper function for cross-type comparisons.

# Unresolved questions
[unresolved]: #unresolved-questions

* Is `PartialOrd` between float and int values as bad idea as it seems?
* Which platforms might need branching in signed-unsigned comparison implementation?
