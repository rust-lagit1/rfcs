- Feature Name: delegation_of_implementation
- Start Date: 2015-12-12
- RFC PR: 
- Rust Issue: 

# Summary
[summary]: #summary

Provide a syntactic sugar to automatically implement a given trait `Tr` using a pre-existing type implementing `Tr`. The purpose is to improve code reuse in rust without damaging the orthogonality of already existing concepts or adding new ones.

# Motivation
[motivation]: #motivation

Let's consider some existing pieces of code:
```rust
// from rust/src/test/run-pass/dropck_legal_cycles.rs
impl<'a> Hash for H<'a> {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.name.hash(state)
    }
}
```
```rust
// from servo/components/devtools/actors/timeline.rs
impl Encodable for HighResolutionStamp {
    fn encode<S: Encoder>(&self, s: &mut S) -> Result<(), S::Error> {
        self.0.encode(s)
    }
}
```
We can see a recurring pattern where the implementation of a method only consists in applying the same method to a subfield or more generally to an expression containing `self`. Those are examples of the well known [composition pattern][object_composition]. It has a lot of advantages but unfortunately it also implies writing explicit boilerplate code again and again. In a classical oop language we could also have opted for inheritance for similar cases. Inheritance comes with its own bunch of problems and limitations but at least it allows a straightforward form of code reuse: any subclass implicitly imports the public methods of its superclass(es).

Rust has no inheritance (yet) and as a result composition is an even more interesting pattern for factoring code than in other languages. In fact it is already used in many places. Some (approximate) figures:

Project           | Occurences of "delegating methods" |
------------------| ---------------------------------- |
rust-lang/rust    | 845                                |
rust-lang/cargo   | 38                                 |
servo/servo       | 314                                |

It would be an improvement if we alleviated the composition pattern so that it can remain/become a privileged tool for code reuse while being as terse as the inheritance-based equivalent. Related discussions:
* [pre-rfc][pre_rfc]
* [some related reddit thread][comp_over_inh] (I didn't participate in)

[pre_rfc]: https://internals.rust-lang.org/t/syntactic-sugar-for-delegation-of-implementation/2633
[comp_over_inh]: https://www.reddit.com/r/rust/comments/372mqw/how_do_i_composition_over_inheritance/
[object_composition]: https://en.wikipedia.org/wiki/Composition_over_inheritance

# Detailed design
[design]: #detailed-design

## Syntax example

Let's add a syntactic sugar so that the examples above become:
```rust
impl<'a> Hash for H<'a> {
    use self.name;
}
```
and
```rust
impl Encodable for HighResolutionStamp {
    use self.0;
}
```

Again this feature adds no new concept. It just simplifies an existing code pattern. However it is interesting to understand the similarities and differences with inheritance. The *delegating type* (`H<'a>` in the first example) implicitely "inherits" methods (`hash`) of the *delegated trait* (`Hash`) from the *surrogate type* (`&'static str` which is the type of the *delegating expression* `self.name`) like a subclass inherits methods from its superclass(es). A fundamental difference is that the delegating type is not a subtype of the surrogate type in the sense of Liskov. There is no external link between the types. The surrogate may even be less visible than the delegating type. Another difference is that the developer has a total control on which part of the surrogate type to reuse whereas class hierarchy forces him/her to import the entire public interface of the superclass (this is because a superclass plays two roles: the role of the surrogate type and the role of the delegated trait).

Other syntaxes have been proposed, including:
```rust
impl<'a> Hash for H<'a> use self.name {}

#[delegate(self.name)]
impl<'a> Hash for H<'a> {}

#[delegate(field = name)]
impl<'a> Hash for H<'a> {}

// on the type itself
struct H<'a> {
    #[delegate(Hash)]
    name : &'static str,
    …
}
```

## Partial delegation

If we consider this piece of code:
```rust
// from rust/src/libsyntax/attr.rs
impl AttrMetaMethods for Attribute {
    fn check_name(&self, name: &str) -> bool {
        let matches = name == &self.name()[..];
        if matches {
            mark_used(self);
        }
        matches
    }
    fn name(&self) -> InternedString { self.meta().name() }
    fn value_str(&self) -> Option<InternedString> {
        self.meta().value_str()
    }
    fn meta_item_list(&self) -> Option<&[P<MetaItem>]> {
        self.node.value.meta_item_list()
    }
    fn span(&self) -> Span { self.meta().span }
}
```
we can identify the recurring expression `self.meta()` but 2 of the 5 methods are more complex. This heterogeneity can be handled simply if we allow partial delegation like in:
```rust
impl AttrMetaMethods for Attribute {

    use self.meta() for name, value_str, span;

    fn check_name(&self, name: &str) -> bool {
        let matches = name == &self.name()[..];
        if matches {
            mark_used(self);
        }
        matches
    }
    fn meta_item_list(&self) -> Option<&[P<MetaItem>]> {
        self.node.value.meta_item_list()
    }
}
```
Only specified methods are automatically implemented with the syntax:
```rust
use delegating_expression for list_of_delegated_methods;
```
For other methods the developer can provide a custom implementation.

### Mixed partial delegation

Partial delegation can naturally be extended to an even more powerful pattern: delegating to different surrogate types depending on the method.
```rust
trait Tr {
    fn do_something_1( … );
    fn do_something_2( … );
    fn do_something_3( … );
    fn do_something_4( … );
}

impl Tr for A { … }
impl Tr for B { … }

struct C { a: A, b: B, … }

impl Tr for C {
    // do_something_1 is delegated to surrogate type A
    use self.a for do_something_1;
    
    // do_something_2 and do_something_3 are delegated to surrogate type B
    use self.b for do_something_2, do_something_3;
    
    // do_something_4 is directly implemented
    fn do_something_4( … ) { … }
}
```
In this pattern, multiple delegated expressions can be provided with each one declaring a list of methods it is expected to handle. 

## Delegating expressions and the type of `self`

In Rust the actual type of the `self` parameter can vary depending on the method in the trait. It is either `Self`, `&mut Self` or `&Self`. This means that:
* each method locally defines the type of `self`;
* for a delegating expression to be valid it must type-check with all delegated methods.

Because inherited mutability apply to expressions like `self.2` and `self.my_field`, they may be used to handle methods where `self` have different type. Furthermore as [auto-referencing and auto-dereferencing][auto] is implicitely applied on method call, the return type of the delegating expression may be transformed if possible to match the expected type of ```self```. However for more complex delegating expression that contains function or method calls with non-convertible return types, such flexibility may not be available. In this case mixed partial delegation can be used to handle differently the groups of methods with respective parameter `self`, `&self` and `&mut self` if needed.

The proposed rule for validity of a delegating expression `expression` when handling method `my_method` in trait `Trait` is the same as the validity of the plain equivalent code:
```rust
impl Trait for MyType {
    fn my_method(self_parameter, other_parameters… ) {
        expression.my_method(other_parameters… )
    }
    …
}
```
Note that technically nothing prevents `expression` to contain several occurences of `self` or even none.

In case the code above contains a type error, the compiler must provide an error message indicating that method `my_method` from trait `Trait` cannot be delegated using `expression` (for example because `my_method` is expecting a `&mut self` but `expression` is returning a result of type `&MySurrogateType`).

[auto]: https://doc.rust-lang.org/nomicon/dot-operator.html

## Associated items

Associated types and constants (or anything that is not a `fn`) are not implicitely delegated. Whenever a trait declares such items, implementations must define their actual values.

## About delegating and surrogate types

All the examples above deal with structs and delegation to subfields. However no restriction is required. Delegating types and surrogates types might be of any kind (structs, tuples, enums, arrays, lambdas, ...) provided it makes sense. An illustrative example with enums:
```rust
enum HTMLColor { White, Silver, Gray, Black,
	Red, Maroon, Yellow, Olive,
	Lime, Green, Aqua, Teal,
	Blue, Navy, Fuchsia, Purple };

impl Coordinates for HTMLColor {
	fn get_red(&self) -> f32 { … }
	fn get_green(&self) -> f32 { … }
	fn get_blue(&self) -> f32 { … }
	fn get_hue(&self) -> f32 { … }
	fn get_saturation(&self) -> f32 { … }
	fn get_brightness(&self) -> f32 { … }
}

enum ThreeBitColor { Black, Blue, Green, Cyan,
	Red, Magenta, Yellow, White };

fn to_html_color(color: &ThreeBitColor) -> HTMLColor { … }

impl Coordinates for ThreeBitColor {
    use to_html_color(&self);
}
```

## Possible extensions

Delegation can be extended in a certain number of directions to handle other specific cases. Propositions below are simply invitation for debate and are not part of the main proposition.

### Delegation for other `Self` parameters 

If method `my_method` contains other parameters with type `Self`, `&Self` or `&mut Self`, delegation may still be possible provided that the delegating expression is simultaneously compatible with all those types. In this case for each such parameter `p` the delegating expression can be transformed by replacing all occurences of `self` by `p`. The resulting expression is then inserted at the position where parameter `p` is normally expected. For example
```rust
// from rust/src/libcollections/btree/map.rs
impl<K: PartialOrd, V: PartialOrd> PartialOrd for BTreeMap<K, V> {
    fn partial_cmp(&self, other: &BTreeMap<K, V>) -> Option<Ordering> {
        self.iter().partial_cmp(other.iter())
    }
}
```
could become 
```rust
impl<K: PartialOrd, V: PartialOrd> PartialOrd for BTreeMap<K, V> {
    use self.iter();
}
```
where the delegating expression `self.iter()` is implicitely transformed into `other.iter()` in order to handle the second parameter of method `partial_cmp`.

### Inverse delegating expressions

Can we handle cases as this one
```rust
// from servo/components/layout/block.rs
impl<T: Clone> Clone for BinaryHeap<T> {
    fn clone(&self) -> Self {
        BinaryHeap { data: self.data.clone() }
    }

    fn clone_from(&mut self, source: &Self) {
        self.data.clone_from(&source.data);
    }
}
```
where `Self` is used as a return type? Yes but we need a second expression for that.
```rust
impl<T: Clone> Clone for BinaryHeap<T> {
    use self.data, BinaryHeap { data: super };
}
```
Here the `super` keyword corresponds to an instance of the surrogate type. It is the symmetric of `self`. The whole expression must have type `Self`. Both direct and inverse delegating expressions may be given at the same time or possibly just one of them if only one conversion is needed.

### Combined delegation

It would be nice if delegation could be combined for multiple traits so that
```rust
// from cargo/src/cargo/core/package_id.rs 
impl PartialEq for PackageId {
    fn eq(&self, other: &PackageId) -> bool {
        (*self.inner).eq(&*other.inner)
    }
}
impl PartialOrd for PackageId {
    fn partial_cmp(&self, other: &PackageId) -> Option<Ordering> {
        (*self.inner).partial_cmp(&*other.inner)
    }
}
impl Ord for PackageId {
    fn cmp(&self, other: &PackageId) -> Ordering {
        (*self.inner).cmp(&*other.inner)
    }
}
```
could be reduced to:
```rust
impl PartialEq + PartialOrd + Ord for PackageId {
    use &*self.inner;
}
```

### Trait-free delegation

Sometimes implementations are trait-free but the same pattern is found like in
```rust
// from rust/src/librustc/middle/mem_categorization.rs
impl<'t, 'a,'tcx> MemCategorizationContext<'t, 'a, 'tcx> {
    
    …
    
    fn node_ty(&self, id: ast::NodeId) -> McResult<Ty<'tcx>> {
        self.typer.node_ty(id)
    }
}
```
Here we have no trait to delegate but the same method signatures are reused and semantically the situation is close to a trait-based implementation. A simple possibility could be to introduce a new trait. An alternative is to allow delegation even without traits but with the name of the method becoming mandatory:
```rust
impl<'t, 'a,'tcx> MemCategorizationContext<'t, 'a, 'tcx> {
    use self.typer for node_ty;
}
```

### Renaming delegation

Maybe the method you want to delegate does not come from the same trait, it also has a distinct original name but it just happens to have the same signature.

```rust
impl A { 
    fn do_something(&mut self, predicate: P) -> Option<i32> where P: FnMut(&str) -> bool { … }
    fn do_something_else(&self) -> i32 { … }
    …
}

trait Tr {
    fn select_first(&mut self, predicate: P) -> Option<i32> where P: FnMut(&str) -> bool;
    fn count(&self) -> i32;
    …
}

fun to_a(b : &B) -> &A { … }

impl Tr for B {
    // here we are mapping distinct methods with same signatures:
    // B::select_first <- A::do_something
    // B::count <- A::do_something_else
    use do_something, do_something_else in to_a(self) for select_first, count;
    …
}
```
The general syntax becomes:
```rust
use list_of_surrogate_methods in delegating_expression for list_of_delegated_methods;
```
and the previous syntax is now just a shortcut for:
```rust
use list_of_delegated_methods in delegating_expression for list_of_delegated_methods;
```

### More complex delegation

`Self` can also appear inside more complex parameter/result types like `Option<Self>`, `Box<Self>` or `&[Self]`. When we have higher-kinded types in Rust a partial solution based on [functor types][functors] may been possible.

[functors]: https://wiki.haskell.org/Functor

### Value-dependent surrogate type

Let's consider a new example:
```rust
enum TextBoxContent { Number(f64), String(&'static str) }

impl Hash for TextBoxContent {
    use ??? ; // how to delegate?
}
```
It seems that in theory we should be able to delegate meaningfully given that for any value of `TextBoxContent` there is an obvious existing implementation for `Hash`. The problem is we cannot select a **single** surrogate type. The actual surrogate type should indeed be chosen based on the runtime value of `Self`. To handle this case I slightly modify the delegation syntax by using a variation of blaenk's proposition: `impl Tr for B { use delegatingExpression.impl; }`. Now this new syntax could be extended to solve our current issue:
```rust
impl Hash for TextBoxContent {
    use (match self { Number(n) => n.impl, String(s) => s.impl });
}
```
Here the delegating expression can contain several branches that does not need to unify from a type perspective. The `.impl` syntax should be replaced by a call to the actual delegated method. Note that although this pattern may occur naturally with enums it can again apply to any kind of types:
```rust
impl Tr for BStruct {
    use (if self.condition { self.field1.impl } else { self.field2.impl });
}
```
However this kind of delegation for value-dependent surrogate types has a limitation: it does not work for methods with multiple `Self` parameters. Indeed there is no guarantee the runtime values for different parameters will select the same branch and then define a consistent surrogate.

# Drawbacks
[drawbacks]: #drawbacks

* It creates some implicit code reuse. This is an intended feature but it could also be considered as dangerous. Modifying traits and surrogate types may automatically import new methods in delegating types with no compiler warning even in cases it is not appropriate (but this issue is the same as modifying a superclass in OOP).
* The benefit may be considered limited for one-method traits.

# Alternatives
[alternatives]: #alternatives

## OOP inheritance

As mentioned before, inheritance can handle similar cases with the advantage its concepts and mechanisms are well known. But with some drawbacks:
* Multiple inheritance is possible but to my knowledge no serious proposition has been made for Rust and I doubt anyone wants to end up with a system as complex and tricky as C++ inheritance (whereas delegation is **naturally multiple delegation**)
* As said before inheritance mixes orthogonal concepts (code reuse and subtyping) and does not allow fine grain control over which part of the superclass interface is inherited.

## Multiple derefs

Some people noticed a similarity with trait `Deref`. A main limitation is that you can only deref to a single type. However one could imagine implementing multiple derefs by providing the target type as a generic parameter (`Deref<A>`) rather than as an associated type. But again you can find limitations:
* As for inheritance visibility control is impossible: if `B` can be derefed to `A` then the entire public interface of `A` is accessible.
* `Deref` only offers a superficial similarity. If `A` implements trait `Tr`, instances of `B` can sometimes be used where `Tr` is expected but as a counter example a `&[B]` slice is not assignable to `fn f(t: &[T]) where T : Tr`. Derefs do not interact nicely with bounded generic parameters.

## Compiler plugin

I was suggested to write a compiler plugin. But I was also told that [type information is not accessible][type_information] (unless you can annotate the delegated trait yourself, which implies you must own it). The problem is that delegation requires to analyse the signatures of methods to be delegated and this is as far as I know not currently possible.

[type_information]: http://stackoverflow.com/questions/32641466/when-writing-a-syntax-extension-can-i-look-up-information-about-types-other-tha

## Do nothing

In the end, it is a syntactic sugar. It just improves the ease of expression, not the capacity to express more concepts. Some simple cases may be handled with deref, others with trait default methods.

One of my concerns is that the arrival of inheritance in Rust may encourage bad habits. Developers are lazy and DRY principle dissuades them from writing repetitive code. The temptation may be strong to overuse inheritance in situations where only code reuse is required (resulting in unnecessary subtyping hierarchy and uncontrolled interface exposure).

# Unresolved questions
[unresolved]: #unresolved-questions

The exact syntax is to be discussed. The proposed one is short but does not name the surrogate type explicitly which may hurt readability. As mentioned before plenty of alternate syntaxes have been proposed. I have personally no strong preference. However I would prefer we first agree on the scope and validity rules of delegation before bikeshedding about the exact final syntax.