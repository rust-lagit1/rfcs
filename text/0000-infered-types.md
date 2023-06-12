- Feature Name: Inferred Types
- Start Date: 2023-06-06
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)


# Summary
[summary]: #summary

This RFC introduces a feature allowing the base type of enums and structs to be inferred in contexts where strict typing information can be exist. Some examples of strict typing include match statements and function calls. The syntax is `_::EnumVariant` for enums, `_ { a: 1 }` for constructing structs, and `_::method()` for impls and traits wher `_` is the type.


# Motivation
[motivation]: #motivation

Rust's goals include clean syntax, that comes with a consice syntax and features like macros making it easier to not repeat yourself. Having to write a type every time you want to do something can be very annoying, repetetive, and not to mention messy. This is a huge problem especialy in large projects with heavy dependency on enums. Additionaly, with large libraries developers can expect to import upwords from 3 traits, structures, and enums. One way developers have solved this is by importing everything from specific modules like [`windows-rs`](https://github.com/microsoft/windows-rs). This is problematic because at a glance, it can not be determined where a module comes from. It can be said that developers need a low compromise solution to solve the problem of large imports and messy code. The intent of this RFC’s is to create something that developer friendly yet still conforming to all of rust's goals.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When crating a struct or enum, infered types can simplify the type into just a underscore. It is important to note, however, that they do not work when the type is not specific enough to be infered like: type parameters. Below are some examples where they do and don't work.

Function calls (structs):
```rust
fn my_function(data: MyStruct) { /* ... */ }

// my_function(MyStruct {
//     value: 1
// });
my_function(_ {
   value: 1
});
```

Function calls (impl):
```rust
fn my_function(data: MyStruct) { /* ... */ }

// my_function(MyStruct::new()});
my_function(_::new());
```

Function returns (enum):
```rust
fn my_function() -> MyEnum {
   // MyEnum::MyVarient
   _::MyVarient
}
```

Match arms:
```rust
enum Example {
   One,
   Two
}

fn my_fn(my_enum: Example) -> String {
   match my_enum {
      _::One => "One!",
      _::Two => "Two!"
   }
}
```

It is important to note that `_` only represents the type; if you have (for example) another enum that can be coerced into the type, you will need to specify it manually. Additionally, any traits required to call an impl will still have to be imported.

```rust
fn my_function(data: MyStruct) { /* ... */ }


my_function(MyStruct2::do_something().into()); // ✅


my_function(_::do_something().into()); // ❌ error[E0599]: variant or associated item not found in `MyStruct`
```


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The underscore operator infers the type of a stuct or enum where there is enough type information in a context in order to infer an exact type. In a statement like `_::Variant`, you can imagine the underscore to be the type. That means that anything you could do with an actual type like `MyEnum::Variant` will still apply in an infered enum. Ultimately, the enum or struct doesn't need to be imported but, traits and other specfied things will need to be imported.

Due to how the rust compiler currently works, lots of changes will need to be made to allow paths to be infered in an order that allows for all of the mentioned. One issue is getting the type of a given context mentioned in [rust-lang/rust#8995](https://github.com/rust-lang/rust/issues/8995).

Finally, here are some examples of non-strict typings that can not be allowed.
```rust
fn convert<T>(argument: T) -> Example {/* ... */}

do_something(convert(_::new()))
//                   ^^^^^^^^ Cannot infer type on generic type argument
```

However, ones where a generic argument can collapse into strict typing can be allowed. The below works because `T` becomes `Example`. This wouldn’t work if `do_something`, however, took a generic.
```rust
fn do_something(argument: Example) {/* ... */}
fn convert<T>(argument: T) -> T {/* ... */}

do_something(convert(_::new()))
```


# Drawbacks
[drawbacks]: #drawbacks

In the thread [[IDEA] Implied enum types](https://internals.rust-lang.org/t/idea-implied-enum-types/18349), many people had a few concerns about this feature. 

These RFCs could create bugs. An example of this is if a function changes has two enum parameters that share common variant names. Because it’s implied, it would still compile with this bug createing unintended behavior wharas by specifying the type names, the compiler would thrown an error.
```rust
enum RadioState {
   Disabled,
   Enabled,
}

enum WifiConfig {
   Disabled,
   Reverse,
   Enabled,
}

fn configure_wireless(radio: RadioState, wifi: WifiConfig) { /* ... */ }
```

Another issue with this is that the syntax `_::` could be mistaken for `::` meaning crate.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There have been many ideas for what this operator should be including `::` add `.`. Despite that, the underscore is the best because it has already been used to infer lifetimes. Additionally the underscore by itself can be used to construct a struct creating a consistent experience. Maintainers should accept this proposal because it can simplify writing Rust code and prevent the large problem of reputition in switch statements. 


# Prior art
[prior-art]: #prior-art


Apple’s Swift had enum inference since 2014 and is not used in most swift codebases with no issues. One thing people have noticed, though, is that it could be used for so much more! That was quite limited and in creating a rust implementation, people need to extend what swift pioneered and make it more universal. That is why this RFC proposes to make the underscore a general operator that can be used outside the small use case of enums and allow it to be used in structs.

# Unresolved questions
[unresolved-questions]: #unresolved-questions


The implementation of this feature still requires a deep dive into how exactly the compiler should resolve the typings to produce the expected behavior, however, algorithems for finding paths for do an already exist.

# Future possibilities
[future-possibilities]: #future-possibilities


I can’t think of anything