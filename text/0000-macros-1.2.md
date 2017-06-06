- Feature Name: macros-1.2
- Start Date: 2017-02-20
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

Stabilize function procedural macros (whose usage looks like `foo!(...)`),
like this was done in “[Macros 1.1]” for custom `derive`,
before “[Macros 2.0]” is fully ready.

[Macros 1.1]: https://github.com/rust-lang/rfcs/blob/master/text/1681-macros-1.1.md
[Macros 2.0]: https://github.com/rust-lang/rfcs/blob/master/text/1566-proc-macros.md


# Motivation
[motivation]: #motivation

The full design of Macros 2.0 has many details (around hygiene, the `TokenStream` API, etc.)
that will require a significant amount of work before it can be fully stabilized.

With Macros 1.1, we chose to stabilize a very small part of the new API
that was nevertheless enough to unlock a significant portion of the benefits.
This RFC propose what is comparatively a small additional step,
while also enabling new use cases.

At the moment, like they used to for custom derive, some crates resort to [complicated schemes]
that involve parsing entire source files with the `syn` crate,
manually expanding a macro, and using the generated file through `include!()`.
This approach is viable (if inconvenient) within one crate for one source file,
but is probably not acceptable for having a library provide a procedural macro
to be used in other projects.

With this RFC accepted and implemented,
libraries running on Rust’s stable channel would be able to export procedural macros
that are as convenient to use as custom derive is since Rust 1.15.

While the use cases for this may not be as prevalent or high-profile as Serde or Diesel,
the additional amount of details being stabilized
(compared to what is already stable with Macros 1.1)
is also very small.

[complicated schemes]: https://github.com/servo/html5ever/blob/e29d495c94/macros/match_token.rs


# Detailed design
[design]: #detailed-design

As a reminder, Macro 1.1 stabilized a new `proc_macro` crate with a very small public API:

```rust
pub struct TokenStream { /* private */ }
impl fmt::Display for TokenStream {}
impl FromStr for TokenStream {
    type Err = LexError;
}
pub struct LexError { /* private */ }
```

As well as an attribute for defining custom derives:

```rust
#[proc_macro_derive(Example)]
pub fn example(input: TokenStream) -> TokenStream {
    // ...
}
```

Until more APIs are stabilized for `TokenStream`,
procedural macros are expected to serialize it to a string
and parse the result, for example with the [syn](https://github.com/dtolnay/syn) crate.

This RFC does *not* propose any such API.
It propose prioritizing the implementation and stabilization
of function procedural macros, that are defined like this:

```rust
#[proc_macro]
pub fn foo(input: TokenStream) -> TokenStream {
    // ...
}
```

And used (in a separate crate that depends on the previous one) like this:

```rust
foo!(...);
foo![...];
foo!{...}
```

The plan to do this eventually has already been accepted as part of Macros 2.0.
This RFC is about prioritization.


# How We Teach This
[how-we-teach-this]: #how-we-teach-this

The [Procedural Macros](https://doc.rust-lang.org/book/procedural-macros.html) chapter of the book
will need to be extended,
as well as the [Procedural Macros](https://doc.rust-lang.org/reference.html#procedrual-macros)
and [Linkage](https://doc.rust-lang.org/reference.html#linkage)
(where it mentions `--crate-type=proc-macro`) sections of the reference.

Terminology:

* *Function procedural macro*: a function declared with the `proc_macro` attribute.
* *Attribute procedural macro*: a function declared with the `proc_macro_attribute` attribute.
* *Derive procedural macro*: a function declared with the `proc_macro_derive` attribute.
* *Procedural macro*: any of the above, unless context disambiguates.
* *Plugin*, *compiler plugin* (preferred), or *syntax extension*:
  a plugin registered with legacy `plugin_registrar` system.


# Drawbacks
[drawbacks]: #drawbacks

As always, stabilizing something means we can’t change it anymore.
However, the risk here seems limited.


# Alternatives
[alternatives]: #alternatives

Don’t prioritize this over the rest of Macros 2.0,
leaving use cases unmet without requiring the Nightly channel
or complex build scripts at each use site.


# Unresolved questions
[unresolved]: #unresolved-questions

In the example above, RFC 1566 [suggests] that the input to `foo` would be the same
for all three calls: such that `input.to_string() == "..."`,
with no way to tell which kind of braces was used to delimit the macro’s input at the call site.

Perhaps that’s fine. There is no way to tell with `macro_rules!` either.
If we do want to make that information available,
it is possible to extend `#[proc_macro]` in the future
to also accept functions that take an additional argument:

```rust
extern crate proc_macro;
use proc_macro::{TokenStream, Delimiter};

#[proc_macro]
pub fn foo(braces_kind: Delimiter, input: TokenStream) {
    // ...
}
```

The `Delimiter` is part of the tokens API [proposed] in RFC 1566:

```rust
pub enum Delimiter {
    None,
    Brace,  // { }
    Parenthesis,  // ( )
    Bracket,  // [ ]
}
```

(The `None` variant would not be used in this case.)

[suggests]: https://github.com/rust-lang/rfcs/blob/master/text/1566-proc-macros.md#detailed-design
[proposed]: https://github.com/rust-lang/rfcs/blob/master/text/1566-proc-macros.md#tokens
