- Feature Name: variant_types
- Start Date: 2016-01-07
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Make enum variants first-class types. Variant types can be used just like any
other type. When a new instance of an enum is created, or we use `@` syntax in a
match expression to create a variable know to be a particular variant, we choose
between the enum and variant type in a similar way to the treatment of integers.
We default to the enum type to preserve backwards compatability.

This RFC previously included a proposal for untagged enums as a kind of union
data type. That has been removed.


# Motivation

Enums are a convenient way of dealing with data which can be in one of many
forms. When dealing with such data, it is typical to match, then perform some
operations on the interior data. However, in many cases there is a large amount
of processing to be done. Ideally we would factor that out into a function,
passing the data to the function. However, currently in Rust, enum variants are
not types and so we must choose an unsatisfactory work around - we pass
each field of the variant separately (leading to unwieldy function signatures
and poor maintainability), we pass the whole variant with enum type (and have to
match again, with `unreachable!` arms in the function), or we embed a struct
within the variant and pass the struct (duplicating data structures for no good
reason). It would be much nicer if we could refer to the variant directly in the
type system.


# Detailed design

Consider the example enum `Foo`:

```rust
pub enum Foo {
    Variant1,
    Variant2(i32, &'static str),
    Variant3 { f1: i32, f2: &'static str },
}
```

We create new instances by constructing one of the variants. The only type
introduced is `Foo`. Variant names can only be used in patterns and for creating
instances. E.g.,

```rust
fn new_foo() -> Foo {
    Foo::Variant2(42, "Hello!")
}
```

This RFC proposes allowing the programmer to use variant names as types, e.g.,

```rust
fn bar(x: Foo::Variant2) {}
struct Baz {
    field: Foo::Variant3,
}
```


## Constructors

Consider `let x = Foo::Variant1;`, currently `x` has type `Foo`. In order to
preserve backwards compatibility, this must remain the case. However, it would
be convenient for `let x: Foo::Variant1 = Foo::Variant1;` to also be valid.

The type checker must consider multiple types for an enum construction
expression - both the variant type and the enum type. If there is no further
information to infer one or the other type, then the type checker uses the enum
type by default. This is analogous to the system we use for integer fallback or
default type parameters.

The type of the variants when used as functions must change. Currently they have
a type which maps from the field types to the enum type:

```rust
let x: &Fn(i32, &'static str) -> Foo = &Foo::Variant2;
```

I.e., one could imagine an implicit function definition:

```rust
impl Foo {
    fn Variant2(a: i32, b: &'static str) -> Foo { ... }
}
```

This would change to accommodate inferring either the enum or variant type,
imagine

```rust
impl Foo {
    fn Variant2<T=Foo>(a: i32, b: &'static str) -> T { ... }
}
```

Since we do not allow generic function types, the result type must be chosen
when the function is referenced:

```rust
let x: &Fn(i32, &'static str) -> Foo = &Foo::Variant2::<Foo>;
let x: &Fn(i32, &'static str) -> Foo::Variant2 = &Foo::Variant2::<Foo::Variant2>;
```

Due to the default type parameter, we remain backwards compatible:

```rust
let x: &Fn(i32, &'static str) -> Foo = &Foo::Variant2;
```

Note that default type parameters on functions have
[recently](https://github.com/rust-lang/rust/pull/30724) been feature-gated for
more consideration. The compiler will only accept referencing a generic function
without specifying type parameters if using the
`default_type_parameter_fallback` feature.


## Matching

When matching an enum, the whole variable can be assigned to a variable using
`@` syntax. Currently such a variable has enum type. With this RFC it would get
the same treatment as newly constructed variants, i.e., it could be inferred to
have either the variant or enum type, with the enum type by default.

Example:

```
fn bar(f: Foo) {
    match f {
        v1 @ Foo::Variant1 => {
            let f: Foo = v1;
        }
        v2 @ Foo::Variant2(..) => {
            let v: Foo::Variant2 = v2;
        }
        _ => {}
    }
}
```

Both branches type check.


## Representation

Enum values have the same representation whether they have enum or variant type.
That is, a value with variant type will still include the discriminant and
padding to the size of the largest variant. This is to make sharing
implementations easier (via coercion), see below.

## Conversions

A variant value may be implicitly coerced to its corresponding enum type (an
upcast). An enum value may be explicitly cast to the type of any of its variants
(a downcast). Such a cast includes a dynamic check of the discriminant and will
panic if the cast is to the wrong variant. Variant values may not be converted
to other variant types. E.g.,

```
let a: Foo::Variant1 = Foo::Variant1;
let b: Foo = a; // Ok
let _: Foo::Variant2 = a; // Compile-time error
let _: Foo::Variant2 = b; // Compile-time error
let _ = a as Foo::Variant2; // Compile-time error
let _ = b as Foo::Variant2; // Runtime error
let _ = b as Foo::Variant1; // Ok
```

See alternatives below, it may be better to not support down-casting.


## impls

`impl`s may exist for both enum and variant types. There is no explicit sharing
of impls, and just because as enum has a trait bound, does not imply that the
variant also has that bound. However, the usual conversion rules apply, so if a
method would apply to the enum type, it can be called on a variant value due to
coercion performed by the dot operator.


## Extension - unsized enums

With the above representation, enum variants are the same size as the enum
itself, which is the size of the largest enum plus the discriminant. This makes
conversion between the variant and enum types easy, and should be the default.
However, there are some use cases where it is preferable to have a more minimal
size for variant values. For example, where variants are of wildly different
sizes and where we usually deal with individual variants and rarely the whole
enum.

For such use cases, we could support an `#[unsized]` attribute on the enum. This
affects the representation of the variants: a variant value is not padded, it
still has the discriminant, but there is no padding to the enum size, the value
is the size of the individual variant (plus the discriminant, of course).

The enum type (but not the variant types) are considered unsized. The effect is
that they may not appear in a Rust program by value, only by reference. There is
no 'unsizing information' (c.f., slices or trait objects) so a pointer to an
unsized enum is a regular pointer, not a fat pointer.

Casting/coercion of enum/variant values can still work as before, since we'll
never access the enum and find a different variant.

### Further extension - remove the discriminant

We don't need the discriminant when we have a value with variant type. We can't
have a value with enum type. When we have a pointer with enum type, we could put
the discriminant in the pointer (making it a fat pointer like we use with other
unsized types). This should all work at the expense of some added complexity in
coercion and matching.


# Drawbacks

The proposal is a little bit hairy, in part due to trying to remain backwards
compatible.


# Alternatives

An alternative to allowing variants as types is allowing sets of variants as
types, a kind of refinement type. This set could have one member and then would
be equivalent to variant types, or could have all variants as members, making it
equivalent to the enum type. Although more powerful, this approach is more
complex, and I do not believe the complexity is justified.

We could remove support for casting from enums to variants, relying on matching.


# Unresolved questions

There is some potential overlap with some parts of some proposals for efficient
inheritance: if we allow nested enums, then there are many more possible types
for a variant, and generally more complexity. If we allow data bounds (c.f.,
trait bounds, e.g., a struct is a bound on any structs which inherit from it),
then perhaps enum types should be considered bounds on their variant types.
There are also interesting questions around subtyping. However, without a
concrete proposal, it is difficult to deeply consider the issues here.
