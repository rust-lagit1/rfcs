- Start Date: 2014-04-30
- RFC PR #: 
- Rust Issue #: 

# Summary

NOTE: I've completely re-written this RFC to integrate all the content of my
first 3 comments on the thread. So, if you read this document, please feel free
to ignore my first 3 comments.

This RFC depends on proving the following 4 points:

1. Interior mutability is a fact of life in Rust

2. `&` references do not offer practical immutability

3. `&mut`'s real guarantee is 'non-aliased'; mutability is just a consequence

4. `&mut` implies that `&` is immutable which is confusing and inaccurate

Therefore, since the term `&mut` is both confusing and inaccurate, I propose
that the name should be changed to something that better reflects the actual
guarantees it provides, such as `&only`.

# Motivation

## 1. Interior mutability is a fact of life in Rust

There are multiple types in the Rust libraries that implement interior
mutability. Some examples include `Cell`, `RefCell`, and `Mutex`. Although
these types all have unsafe implementations, they all expose a completely safe
interface. These objects aren't marked in any particular way - the only way to
know if a type implements interior mutability is to read the documentation or
audit the implementation. As a result, its perfectly valid for regular, safe
Rust code to mutate objects using `&` references and the language doesn't
provide any tools to tell when this might be happening. There is nothing
'special' about these types - they are just plain old types. `Cell` and
`RefCell` might qualify as unusual special cases, but `Mutex` is a fairly
fundamental type for concurrent programming in many languages.

## 2. `&` references do not offer practical immutability

As stated above, `&` references can be used by safe code to mutate objects.
This is obly possible if the objects themselves are implemented with unsafe
code. If it is known that an object does not contain unsafe code and that no 
other object that is transitively reachable through it contains a unsafe code 
either, then it is impossible to mutate the object through a `&` reference.
This doesn't provide a particularly practical guarantee for the programmer, 
however.

Lets look at the following code:

```rust
use some_library::Flag;

fn do_something(holder: &Flag) {
// ... 
}

fn main() { 
    let do_calc = Flag::new(true);
    if do_calc.is_true() {
        println!("Starting calculation...");
    }
    do_something(&do_calc);
    if do_calc.is_true() {
        println!("Calculation completed!");
    }
}
```

The program always prints out a "Starting" message. Then, it does a calculation
of some sort. If a "Starting" message is printed, but no "Completed" message
is, users will clearly see this as a bug. Will this program always print out
"Completed"?

First things first - what does the `Flag` type do? Well, its a type I just made
up. All I'll tell you is that it holds a `bool` value and that one of the
methods it provides is `get()` which returns the contained value. Furthermore,
I'll let you assume that the `Flag` type is implemented perfectly and that the
entire program is free from undefined behavior.

So, given some of these fairly generous assumptions, what is the answer? The
real answer is that there is no way to tell based on what I've told you since
you have no way of knowing if the `Flag` type implements interior mutability.

Is the question unfair? Maybe, but this scenario I think prettly closely
reflects what its like to work on a new code base, or with a new library, or
any other scenario where you aren't intimately familiar with every line of code
in every file - ie, most of the time.

How do you determine if `Flag` implements interior mutability? You have to
consult its documentation and/or audit its code to know since the compiler
won't tell you. After you've done that, whenever the implementation of `Flag`
changes in any way, you then need to do that again. This is not practical
for even medium scale projects.

So, my definition of "practical immutability" is: the ability to locally reason
(ie, looking at just a single function at a time) about the mutability of a
variable. A `&` refence clearly does not provide this.

Since `&` does not offer "practical immutability", I further argue that
`&` is no better than `&mut` when you are trying to reason about
mutability. It is true that most of the time objects cannot be mutated through 
a `&`. However, I would argue that any guarantee that needs to include the
phrase "most of the time" is not a very good guarantee.

## 3. `&mut`'s real guarantee is 'non-aliased'; mutability is just a consequence

Let's examine the code below:

```rust
use std::cell::Cell;

fn some_func(c: &Cell<uint>) { ... }

fn funca(c1: &Cell<uint>, c2: &Cell<uint>) -> uint {
    some_func(c2);
    c1.get()
}

fn funcb(c1: &mut Cell<uint>, c2: &Cell<uint>) -> uint { 
    some_func(c2);
    c1.get()
}
```

This time, we have two functions that pass their 2nd parameter to another
function and then return the value contained in their 1st parameter.
Furthermore, both functions are identical, except that `funca` accepts both of
its parameters as `&Cell`s while `funcb` accepts its first parameter as a
`&mut Cell`. Unlike with the previous example, we use a real type, `Cell`,
whose code can be inspected to see exactly what it does.

The caller of both of these functions expects them to return the value
originally contained in the first parameter. Does `funca` do this?

Just like before, there is no way to know. What `funca` returns depends on:

* If `c1` and `c2` reference the same object.

* If `Cell` implements interior mutability (It does).

* What `some_func` does.

The caller might be defined in another file. We know that `Cell` implements
interior mutability. However, if it were some other type, we couldn't be sure
unless we audited its code, which might also be in another file. Finally,
`some_func` might be defined in another file. So, the answer to the fairly
simple question above depends on analyzing code in many different locations,
some of which you may not have source code for.

Does `funcb` work as expected?

Yes.

That answer is quite a bit more straight forward and importantly only involves
local reasoning about `funcb` itself and not its caller or the function it
calls.

An important thing to note is that at no point in either of the functions is a
function called that takes a `&mut`. Furthermore, we're interested in what 
`c1.get()` results in, but the confounding variable are the effects of 
`some_func(c2)`. The answer for `funcb` is so much simpler not because the 
`&mut` reference used to pass in `c1` makes any direct guarantees about `c1`
but rather because it makes a guarantee about the relationship between `c1` and
`c2` - specifically that they do not reference the same object.

`&mut` is sometimes said to provide a guarantee that a value may be mutated.
Other times, it is said that it provides a guarantee of being non-aliased.
I argue that one of those things must be the basic guarantee provided by the
type and that the other concept must arrise from that basic guarantee.

Due to the example above, I argue that the basic guarntee of `&mut` is one of
being non-aliased and that mutability arrises from this guarantee. It does not
make sense to say that the guarantee is one of mutability since what we see in
this example is specifically a guarantee of *non-mutability* - that `c1` cannot
be mutated through `c2`.

## 4. `&mut` implies that `&` is immutable which is confusing and inaccurate

Its not hard to find blog posts, email threads, reddit discussions, etc. where
`&` is referred to as immutable. I believe I've laid out the case that this is
incorrect.

You can also find various suggestions to require a "mut" keyword in order to
pass a `&mut` reference to a function that also expects a `&mut` reference.
This seems to make sense at first blush - I'm ok with passing a `&mut` to a
function that expects a `&` since it feels like I can be sure that it won't
mutate it. However, then I become worried that function will change someday to
accept a `&mut`. So, what I want is a compiler warning if the function I'm
calling changes and starts mutating my variable without me expecting it. As
I've described already, however, `&` doesn't mean immutable, so this request,
while good intentioned, doesn't really accomplish the goal of controlling
mutation. Lets say that `&mut` were changed to something (such as `&only`)
which relates to aliasability. I would wager that these discussions will go
away. It wouldn't make sense to require an annotation to re-affirm that a 
non-aliased reference is still non-aliased when passing it to another function.

This type of confusion isn't good for the lanauge. The basic pointer types
should be well understood, but I would argue that there is widespread confusion
of exactly which guarantees each pointer type makes. The biggest point of
confusion being the idea that Rust supports immutable references, which I argue
it does not. (Granted, its very hard to define or prove "widespread"). With
`&mut` references present in the language, this misunderstanding will never
go away - no amount of documentation to the contrary will ever overcome
opening up a file, seeing `&mut`, and then concluding that `&` must be
immutable, since if it wasn't, why would `&mut` exist?

## Conclusion

Due to all the reasons above, it is my oppinion that `&` is not practically
immutable, `&mut`'s basic guarantee is one of aliasability not mutability, and
that the current naming is causing quite a bit of confusion and will continue 
to do so into the future. Therefore, I propose that `&mut` be renamed to 
`&only` which much more closely reflects the gurantee it provides.

This means that Rust no longer has an "immutable reference" type and a "mutable
reference" type. Instead it has an "aliased reference" type and a "non-aliased
reference type". However, thats really already the case. It would be nice if
there were an immutable reference type - but there isn't - and innaccurate
naming won't make it so.

In closing, lets look back at `funcb` from above. I believe that it looks quite
strange that we pass `c1` as a `&mut Cell` in order to guarantee that it
*won't* be mutated. Let's re-write that example using `&only`:

```rust
fn funcb(c1: &only Cell<uint>, c2: &Cell<uint>) -> uint { 
    some_func(c2);
    c1.get()
}
```

In my oppinion, that is significantly clearer.

# Drawbacks

* This would be a massive, massive change.

* It would require an extra character for non-aliased references.

# Detailed design

Rename `&mut` to `&only`.

# Alternatives

1. Leave things as they are.

2. Implement a deeply immutable reference type or a way to assert that a type
does not implement interior mutability. Neither of these would really address
the issue of `&mut` being confusing though.

# Unresolved questions

* Is `&only` the best name for a non-aliased reference?

