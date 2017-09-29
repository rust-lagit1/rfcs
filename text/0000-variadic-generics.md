- Feature Name: `variadic_generics`
- Start Date: 2017-2-22
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes the addition of several features to support variadic generics:
- An intrinsic `Tuple` trait implemented exclusively by tuples
- `...T` syntax for tuple types where `T: Tuple`
- `let (head, ...tail) = tuple;` pattern-matching syntax for tuples
- `let tuple = (head, ...tail);` syntax for joining an element with a tuple

# Motivation
[motivation]: #motivation

Variadic generics are a powerful and useful tool commonly requested by Rust
users (see
[#376](https://github.com/rust-lang/rfcs/issues/376) and
[#1921](https://github.com/rust-lang/rfcs/pull/1921)). They allow
programmers to abstract over non-homogeneous collections, and they make it
possible to implement functions which accept arguments of varying length and
type.

Rust has a demonstrable need for variadic generics.

In Rust's own standard library, there are a number of traits which have
been repeatedly implemented for tuples of varying size up to length 12 using
macros. This approach has several downsides:
- It presents arbitrary restrictions on tuples of size 13+.
- It increases the size of the generated code, resulting in slow compile times.
- It complicates documentation
(see the list of trait implementations in
[this documentation](https://doc.rust-lang.org/std/primitive.tuple.html)).

These arbitrary tuple-length restrictions, manual tuple macros, and confusing
documentation all combine to increase Rust's learning curve.

Furthermore, community library authors are required to implement similar
macro-based approaches in order to implement traits for tuples. In the `Diesel`
crate, it was discovered that replacing macro-generated tuple implementations
with a structurally-recursive implementation (such as the one proposed here)
resulted in a 50% decrease in the amount of code generated and a 70% decrease
in compile times ([link](https://github.com/diesel-rs/diesel/pull/747)). This
demonstrates that Rust's lack of variadic generics is resulting in a subpar
edit-compile-debug cycle for at least one prominent, high-quality crate.

The solution proposed here would resolve the limitations above by making it
possible to implement traits for tuples of arbitrary length. This change would
make Rust libraries easier to understand and improve the edit-compile-debug
cycle when using variadic code.


# Detailed design
[design]: #detailed-design

## The `Tuple` Trait
The following would be implemented by all tuple types:
```rust
trait Tuple {
    type AsRefs<'a>: Tuple + 'a;
    type AsMuts<'a>: Tuple + 'a;
    fn elements_as_refs<'a>(&'a self) -> Self::AsRefs<'a>;
    fn elements_as_mut<'a>(&'a mut self) -> Self::AsMuts<'a>;
}
```

The types `AsRefs` and `AsMuts` are the corresponding tuples of references to
each element in the original tuple. For example,
`(A, B, C)::AsRefs = (&A, &B, &C)` and
`(A, B, C)::AsMuts = (&mut A, &mut B, &mut C)`

The `Tuple` trait should only be implemented for tuples and marked with the
`#[fundamental]` attribute described in
[the coherence RFC](https://github.com/rust-lang/rfcs/blob/master/text/1023-rebalancing-coherence.md).
This would allow coherence and type-checking to be extended to assume that no
implementations of `Tuple` will be added. This enables an increased level of
negative reasoning making it easier to write blanket implementations of traits
for tuples.

## The `...T` Type Syntax
This syntax would allow for expansion of tuple types into a list of types.
For example, `(A, B, C)` could be represented as `(A, ...(B, C))` or
`(...(A, B), C)`. This allows users to express type lists of varying arity.

## The `...x` Pattern-Matching Syntax
This syntax allows for splitting apart the head and tail of a tuple. For
example, `let (head, ...tail) = (1, 2, 3);` moves the head value, `1`, into
`head`, and the tail value, `(2, 3)`, into `tail`. Similarly, the last element
of a tuple could be matched out using `let (...front, last) = (1, 2, 3);`.

## The `...x` Joining Syntax
This syntax allows pushing an element onto a tuple. It is the natural inverse
of the pattern-matching operation above. For example,
`let tuple = (1, ...(2, 3));` would result in `tuple` having a value of
`(1, 2, 3)`.

## An Example

Using the tools defined above, it is possible to implement `TupleMap`, a
trait which can apply a mapping function over all elements of a tuple:

```rust
trait TupleMap<F>: Tuple {
    type Out: Tuple;
    fn map(self, f: F) -> Self::Out;
}

impl<F> TupleMap<F> for () {
    type Out = ();
    fn map(self, _: F) {}
}

impl<Head, Tail, F, R> TupleMap<F> for (Head, ...Tail)
    where
    F: Fn(Head) -> R,
    Tail: TupleMap<F>,
{
    type Out = (R, ...<Tail as TupleMap<F>>::Out);
    
    fn map(self, f: F) -> Self::Out {
        let (head, ...tail) = self;
        (f(head), ...tail.map(f))
    }
}
```

This example is derived from
[a playground example by @eddyb](https://play.rust-lang.org/?gist=8fd29c83271f3e8744a3f618786ca1de&version=nightly&backtrace=0)
that provided inspiration for this RFC.

The example demonstrates the concise, expressive code enabled
by this RFC. In order to implement a trait for tuples of any length, all
that was necessary was to implement the trait for `()` and `(Head, ...Tail)`.

## Working With References

Since `let (head, ...tail) = tuple;` consumes `tuple`, there is an extra step
required when implementing traits that take `&self` rather than `self`. Previous
RFC proposals have included a conversion from `&(Head, ...Tail)` to
`(&Head, &Tail)`.
Unfortunately, this would require that every tuple `(Head, Tail...)` contain a
sub-tuple `Tail` (i.e. `(A, B, C)` would need to contain `(B, C)`).
This would limit tuple iteration to a single direction and prevent desirable
field-reordering optimizations.

Instead, this RFC proposes the use of `elements_as_refs` and `elements_as_mut`
methods to convert from `&(A, B, C)` to `(&A, &B, &C)` and from `&mut (A, B, C)`
to `(&mut A, &mut B, &mut C)`, respectively.

Using this strategy, `Clone` can be implemented as follows:

```rust
trait CloneRefsTuple: Tuple {
    type Output: Tuple;
    /// Turns `(&A, &B, ...)` into `(A, B, ...)` by cloning each element
    fn clone_refs_tuple(self) -> Self::Output;
}
impl CloneRefsTuple for () {
    type Output = ();
    fn clone_refs_tuple(self) -> Self::Output { }
}
impl<'a, Head, Tail> CloneRefsTuple for (&'a Head, ...Tail)
    where Head: Clone, Tail: CloneRefsTuple
{
    type Output = (Head, <Tail as CloneRefsTuple>::Output);

    fn clone_refs_tuple(self) -> Self::Output {
        let (head, ...tail) = self;
        (head.clone(), tail.clone_refs_tuple())
    }
}

impl<T: Tuple> Clone for T where
    for<'a> T::AsRefs<'a>: CloneRefsTuple<Output=T>
{
    fn clone(&self) -> Self {
        self.elements_as_refs().clone_refs_tuple()
    }
}
```

# How We Teach This
[teach]: #teach

The `...X` syntax can be thought of as a simple syntax expansion. A good mental
tool is to think of `...` as simply dropping the parenthesis around a tuple, e.g.
`...(A, B, C)` expanding to `A, B, C`.

This allows for operations that were previously impossible, such as appending
to the front or tail of a tuple using `(new_head, ...tuple)` or
`(...tuple, new_last)`, or even `(new_head, ...tuple, new_tail)`.

When teaching this new syntax, it is important to note that the proposed system
allows for more complicated matching than traditional `Cons`-lists. For example,
it is possible to match on `(...Front, Last)` in addition to the familiar
`(Head, ...Tail)`-style matching.

The exact mechanisms used to teach this should be determined after getting more
experience with how Rustaceans learn. After all, Rust users are a diverse crowd,
so the "best" way to teach one person might not work as well for another. There
will need to be some investigation into which explanations are more
suitable to a general audience.

As for the `(head, ...tail)` joining syntax, this should be explained as
taking each part of the tail (e.g. `(2, 3, 4)`) and inlining or un-"tupling"
them (e.g. `2, 3, 4`). This is nicely symmetrical with the `(head, ...tail)`
pattern-matching syntax.

The `Tuple` trait is a bit of an oddity. It is probably best not to go too
far into the weeds when explaining it to new users. The extra coherence
benefits will likely go unnoticed by new users, as they allow for more
advanced features and wouldn't result in an error where one didn't exist
before. The obvious exception is when trying to implement the `Tuple` trait.
Attempts to implement `Tuple` should resort in a relevant error message,
such as "The `Tuple` trait cannot be implemented for custom types."

# Drawbacks
[drawbacks]: #drawbacks

As with any additions to the language, this RFC would increase the number
of features present in Rust, potentially resulting increased complexity
of the language.

There is also some unfortunate overlap between the proposed `(head, ...tail)`
syntax and the current inclusive range syntax. There will need to be some
discussion as to how best to overcome this. One possible path could be to
use `tail...` instead, as it's not obvious that `x...` is distinct from
`x..` in a useful way (inclusive vs. exclusive range to infinity).

# Future Extensions
[future]: #future-extensions

## Allow `...T` in Argument Position
The RFC as proposed would allow for varied-length argument lists to functions
in the form of explicit tuples. It would allow variadic functions to be
declared and used like this:

```rust
trait MyTrait: Tuple {...}

impl MyTrait for () { ... }

impl<Head, Tail> MyTrait for (Head, ...Tail) where Tail: Tuple, ... { ... }

fn foo<T: MyTrait>(args: T) { ... }

// Called like this:
foo((arg1, arg2, arg3));
```

However, this approach requires that functions be called with explicit tuple
arguments, rather than the more natural syntax `foo(arg1, arg2, arg3)`.
For this reason, it may be beneficial to allow `foo` to be declared like
`fn foo<T: MyTrait>(...args: ...T)` and called like
`foo(arg1, arg2, arg3)`.

## Allow `...T` in Generic Type Lists
Another possible extension would be to allow `...T` in generic type lists,
e.g. `foo<...T>`. Without this extension, a non-type-inferred call to `foo`
would be written `foo::<(T1, T2, T3)>(arg1, arg2, arg3)`. However, this is
unnecessarily verbose and doesn't allow for existing traits and functions
to support variadic argument lists.

By supporting `...T` in generic type lists, `foo`'s declaration would become
`fn foo<...T>(...args: ...T) where T: MyTrait`, and it could be called like
`foo::<T1, T2, T3>(arg1, arg2, arg3)`.

As mentioned before, this extension would allow functions, traits, and types
to backwards-compatibly support variadic argument lists. For example, the
`Index<T>` trait could be extended to support indexing like
`my_vec[i, j]`:

```rust
pub trait Index<...Idxs> where Idxs: Tuple + ?Sized {
  type Output: ?Sized;
  fn index(&self, ...indices: ...Idxs) -> &Self::Output;
}
```

Similarly, this extension would allow the currently awkward `fn_traits` to use
the syntax
`T: Fn<Arg1, Arg2, Arg3>` instead of the current (unstable) syntax
`T: Fn<(Arg1, Arg2, Arg3)>`.

# Alternatives
[alternatives]: #alternatives

- Do nothing.
- Implement one of the other variadic designs, such as
[#1582](https://github.com/rust-lang/rfcs/pull/1582) or
[#1921](https://github.com/rust-lang/rfcs/pull/1921)
- Include explicit `Head`, `Tail`, and `Cons<T>` associated types in the `Tuple`
trait. This could allow the above syntax to be implemented purely as sugar.
However, this approach introduces a lot of additional complexity. One of the
complications is that such a trait couldn't be implemented for `()`, so
there would have to be separate `Cons` and `Split` traits, rather than one
unified `Tuple`.
- Allow partial borrows or tuple reference splitting. Previous proposals have
included a transformation from `&(Head, Tail...)` to `(&Head, &Tail)`.
However, this would require that every tuple `(Head, Tail...)`
contain a sub-tuple `Tail` (i.e. `(A, B, C)` would need to contain `(B, C)`).
This would limit tuple iteration to a single direction and prevent desirable
field-reordering optimizations.
In order to avoid this restriction, this proposal includes a transformation
from `&(A, B, C)` to a tuple containing references to all the individual types:
`(&A, &B, &C)`. Iteration over tuples is then performed by-value (where the
values are references).

# Unresolved questions
[unresolved]: #unresolved-questions
- Should the `Tuple` trait use separate `TupleRef<'a>` and `TupleMut<'b>` traits
to avoid dependency on ATCs? It seems nicer to have them all together in one
trait, but it might not be worth the resulting feature-stacking mess.
- Should either of the future extensions be included in the initial RFC?
