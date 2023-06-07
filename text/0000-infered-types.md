- Feature Name: Inferred Types
- Start Date: 2023-06-06
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)


# Summary
[summary]: #summary

This RFC introduces a feature allowing the base type of enums and structs to be inferred in certain contexts including but not limited to function calls. The syntax is as follows `_::Variant`.


# Motivation
[motivation]: #motivation

Using large libraries usually requires developers to import large amounts of traits, structures, and enums. To just call a function on an implementation can take upwards of 3 imports. One way developers have solved this is by importing everything from specific modules. Importing everything has its own problems like trying to guess where imports are actually from. Developers need a low compromise solution to solve this problem and that comes in the form of inferred types.

Swift has had a system to infer types since around 2014. Its system has been praised by many developers around the world. It’s system is as follows:
```swift
enum EnumExample {
   case variant
   case variant2
}


example(data: .variant);
```

This RFC’s intent was to create something similar to the system already tried and tested in swift. Additionally, the underscore is already used to imply type’s lifetimes so it runs consistent with the rust theme.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


Implied types basically replace the path to a type with `_`. They can be used in places where strong typing exists. This includes function calls and function returns. You can think about the `_` as expanding into the type’s name.

Function calls (struct):
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
match Example::One {
   _::One => println!("One!"),
   _::Two => println!("Two!")
};
```

It is important to note that `_` only represents the type; if you have (for example) another enum that can be coerced into the type, you will need to specify it manually. Additionally, any traits required to call an impl will still have to be imported.

```rust
fn my_function(data: MyStruct) { /* ... */ }


my_function(MyStruct2::do_something().into()); // ✅


my_function(_::do_something().into()); // ❌ variant or associated item not found in `MyStruct`
```

When developing, an IDE will display the type’s name next to the underscore as an IDE hint similar to implied types. Reading and maintaining code should not be impacted because the function name should give context to the type.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This RFC should not take much to implement as you can think of the `_` as if it was just expanding to a type based on the context and where it’s used. Anywhere with a strongly typed argument as mentioned above can be inferred. This additionally includes let and const statements with type definitions in the left hand side.

Here is an example of what should happen when compiling.
```rust
let var: MyEnum = _::Variant;
```
becomes:
```rust
let var = MyEnum::Variant;
```

One issue is getting the type of a given context mentioned in [rust-lang/rust#8995](https://github.com/rust-lang/rust/issues/8995).

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

These RFCs could create bugs. An example of this is if a function changes has two enum parameters that share common variant names. Because it’s implied, it would still compile with this bug causing UB.
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


Maintainers should accept this proposal because it can simplify writing Rust code. Especially in enum heavy libraries like [windows-rs](https://github.com/microsoft/windows-rs). There have been many ideas for what this operator should be including `::` add `.`. Despite that, the underscore is the best because it has already been used to infer lifetimes. Additionally the underscore by itself can be used to construct a struct creating a consistent experience.


If this RFC doesn’t happen, writing rust code will continue to feel bloated and old.


# Prior art
[prior-art]: #prior-art


Apple’s Swift had enum inference since 2014 and is not used in most swift codebases with no issues. One thing people have noticed, though, is that it could be used for so much more! That was quite limited and in creating a rust implementation, people need to extend what swift pioneered and make it more universal. That is why this RFC proposes to make the underscore a general operator that can be used outside the small use case of enums and allow it to be used in structs.


# Unresolved questions
[unresolved-questions]: #unresolved-questions


A few kinks on this are whether it should be required to have the type in scope. Lots of people could point to traits and say that they should but others would disagree. From an individual standpoint, I don’t think it should require any imports but, it really depends on the implementers as personally, I am not an expert in *this* subject.


# Future possibilities
[future-possibilities]: #future-possibilities


I can’t think of anything