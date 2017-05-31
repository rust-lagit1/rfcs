- Start Date: 2014-03-14
- RFC PR #: (leave this empty)
- Rust Issue #: (leave this empty)

# Summary

Extending the [DST proposal](http://smallcultfollowing.com/babysteps/blog/2014/01/05/dst-take-5/), add the concept of "fat objects" to the language as an alternative to fat pointers.  These are DST objects that carry their extra information (i.e. vtable or size) right before their content, just like C++ objects do.  Despite being dynamically sized, because they carry this extra info with them they can be pointed to by plain old "thin" pointers.  Combining this with any of the proposals for structural inheritance would solve for the requirements that led to the virtual methods proposal, without having to introduce any additional parallel dynamic method dispatch systems.

# Motivation

The following requirements were given in the [virtual methods RFC](https://github.com/rust-lang/rfcs/pull/5):

>Supporting efficient, heterogeneous data structures such as the DOM. Precisely we need a form of code sharing which satisfies the following constraints:

>   1. Cheap field access from internal methods;
>   2. Cheap dynamic dispatch of methods;
>   3. Cheap downcasting;
>   4. Thin pointers;
>   5. Sharing of fields and methods between definitions;
>   6. Safe, i.e., doesn't require a bunch of transmutes or other unsafe code to be usable.

The goal is to meet all of the requirements listed above, while still respecting the following constraints that the addition of `virtual` into the language would violate:

   * Keep only one dynamic method dispatch system in the language, rather than two parallel ones
   * Continue to allow for the logical separation of data layout in memory (structs) from the dynamic method interfaces (traits), even in situations where thin pointers are needed

# Detailed design

## Baseline Assumptions and Overview

This RFC assumes that dynamically sized types, as described in the RFC-as-blog-post [DST, Take 5](http://smallcultfollowing.com/babysteps/blog/2014/01/05/dst-take-5/) by @nikomatsakis, will be added to the language.  Refer to that post for details about this, as it will not be re-described here.  The remainder of this RFC uses "DST, Take 5" as its baseline.

The following features are then added on top of this:

   1. Fat Objects
   2. Single Inheritance
   3. Downcasting

Each of them are fairly orthoganal to each other (so in theory each should be in their own RFC) but only together do they meet the listed requirements.

The focus of this RFC is on the fat objects.  The others are variations on ideas that have been well discussed elsewhere.  But all of them are needed to get to the end goal, so they are incorporated here for illustrative purposes.

## Definitions 

The DST proposal defines a conceptual type-level function called `Fat` that defines how to create a dynamically sized object type from a concrete type, based on the relation `Fat(T as U) = v`, where `v` is the additional data needed to be able to make use of a pointer to `U` constructed from a `T`.  It shows how to create fat pointers `(*U, v)` given that relation.  In the descriptions below, `Fat(T as U) = v` has the same meaning it does in the DST proposal.

## Fat Objects

For any `T`, `U` and `v` such that `Fat(T as U) = v`, it is possible to create a "fat object" which carries the extra word along with it: `(v,T)`.  For traits, this layout is the same as that of a C++ object that starts with its vtable.

Syntactically, this is expressed as if it were a generic struct that can take either one or two type parameters:

   * `Fat<U,T>` denotes a fat concrete object type.  In memory it is laid out as if it were the tuple `(v,T)` for a specific `T` and `U` where `Fat(T as U) = v`.  `T` must be a concrete type, and `U` must be an existential variant of that type.  So either `T` is `[Elem, ..n]` and `U` is `[Elem]`, or `U` is a trait implemented by the concrete type `T`.  `Fat<U,T>` is not dynamically sized, so it can be used anywhere a normal statically sized type can--as a variable or parameter, inside an array, etc.  It implements `Deref<T>` and `DerefMut<T>`, which obtain borrowed pointers to the plain `T` object.  It also has a method `unwrap() -> T` that discards the extra word and leaves a plain `T`.  There is no automatic implementation added such that `Fat<U,T>` implements trait `U`, but auto-deref should allow you to call methods of any trait implemented by `T` (including but not limited to `U`).
   * `Fat<U>`  is a dynamically sized fat existential type.  `U` must be a dynamically sized type.  Just like a plain `U` is a stand-in for any `T` where `T as U`, `Fat<U>` stands in for any `Fat<U,T>` where `T as U`.  Since it is dynamically sized, `Fat<U>` is only usable in the places where other DSTs can be used.  But unlike other DSTs, pointers to `Fat<U>` are plain old thin pointers.  When the object is used, the required extra word can be retrieved from its place right before the object itself.

A fat object can only be fat for one trait, no matter how many traits the type actually implements.

An intrinsic function `fat` is added to `std` to create fat objects:

`fn fat<unsized U,T>(t:T) -> Fat<U,T>`

It is allowed when `U` is a trait implemented by `T`, and when `T` is a fixed length array and `U` is the corresponding unknown-length array of the same type.

The same pointer conversion rules laid out in the DST proposal between `T` and `U` also apply to `Fat<U,T>` and `Fat<U>`.  The only difference is that the pointers to `Fat<U>` are still thin pointers, despite `Fat<U>` being dynamically sized.  So a `&Fat<U,T>` can be automatically converted to a `&Fat<U>`, and these pointers have the exact same representation.

For structs that are themselves dynamically sized because they end with a DST, "fatness" propagates up the chain.  Using the `RcData` example from the DST propsal, `RcData<Fat<U>>` would be considered a fat DST, and therefore the pointer inside a `Rc<Fat<U>>` would be a thin pointer.

### Examples

```rust
struct MyStruct {
    a:int
}
impl MyStruct {
    fn new() -> MyStruct { MyStruct }
    fn bar(&self) { }
}

trait MyTrait {
    fn foo(&self) -> int
}

impl MyTrait for MyStruct {
    fn foo(&self) -> int { self.a }
}

// The size of this struct is the size of 2 pointers, not 4.
struct HasABunchOfPointers<'a>(&'a Fat<MyTrait>, ~Fat<MyTrait>);

struct DynamicallySizedWrapper<unsized U> {
    some_stuff: uint,
    data:U
}

fn myfunc() {
    let thin_owned_trait_pointer = box fat(MyStruct::new()) as ~Fat<MyTrait>;
    let thin_borrowed_pointer = &*thin_owned_pointer;    // inferred type is &Fat<MyTrait>
    thin_borrowed_pointer.foo();  // Dynamically dispatched, uses the vtable found through the pointer like C++
    
    let fat_stack_object = fat::<MyTrait>(MyStruct::new());  // inferred type is &Fat< MyTrait,MyStruct>
    fat_stack_object.foo();  // Sugar for (*fat_stack_object).foo() via autoderef
    
    let plain_object = fat_stack_object.unwrap();

    let also_a_fat_object = DynamicallySizedWrapper{some_stuff:0, data: fat::<MyTrait>(plain_object)};
    let also_a_thin_pointer = &also_a_fat_object as &DynamicallySizedWrapper<Fat<MyTrait>>;
}

```

From the motivating requirements, we now have thin pointers, cheap dynamic dispatch of methods, and cheap field access from internal methods.  This has the same number of pointer indirections as the `virtual fn` proposal, but keeps dynamic dispatch unified in one place: traits.

## Single Inheritance

Single inheritance is necessary to meet all the listed requirements, but is not the focus of this RFC.  Another RFC should focus on choosing the exact form of it.  For the purposes of the examples below I'll assume the version of single inheritance proposed in the @nikomatsakis [blog post from October 2013](http://smallcultfollowing.com/babysteps/blog/2013/10/24/single-inheritance/).  This RFC is not advocating for this over any of the other proposed forms of single inheritance--others, such as [traits with fields](https://github.com/mozilla/rust/issues/9912) or just so I can use it to show that combining it with fat objects meets the requirements that inspired the `virtual struct` proposal.

## Downcasting

An intrinsic function `downcast<Derived,Base>` is added to `std`. It looks like this:

```rust
fn downcast<Derived,Base>(b:Base) -> Option<Derived>;
```

It has some special compiler checks to ensure that at each call site, the type system allows for the possibility that the downcast could succeed.  A call to `downcast` is allowed at compile time if and only if `Derived` is coercible to `Base`.  It succeeds at run time and returns `Some` if the actual run-time type of `b` is coercible to `Derived`.  Otherwise it returns `None`.

This should work for both the fat and thin variants of DST pointers.  The vtable or size can be accessed at run time, either as part of the fat pointer or by dereferencing the thin pointer.

## A Simplified DOM-Like Example

The DOM is how the requirements list came about, so here's a simplified DOM-like tree using the features above.  The numbers in brackets show a place where each requirement from the motivation section is demonstrated:

```rust
struct NodeFields<'a> {
    // Three thin pointers and a vector of thin pointers: [4]
    parent:Option<&'a Fat<Node>>,
    left_sibling:Option<&'a Fat<Node>>,
    right_sibling:Option<&'a Fat<Node>>,
    children: Vec<&'a Fat<Node>>
}
impl NodeFields {
    // Methods defined here are available to anything that extends NodeFields, are statically
    // dispatched, and cannot be overridden
    fn get_grandparent(&self) -> Option<&'a Fat<Node>> {
        self.parent.and_then(|p| p.parent)
    }    
}

trait Node<'a>: NodeFields<'a> {
    // Methods defined here have access to the fields and methods of NodeFields via `self` [5]
    fn frobify_children(&self) {
        // .children is a static offset field access--no extra pointer indirection [1]
        for node in self.children.iter() {
            node.frobify(); // dynamic method dispatch [2]
        }
    }
    
    fn frobify(&self);
}

struct Element<'a>: NodeFields<'a> {
    name: ~str,
}
impl<'a> Node<'a> for Element<'a> {
    fn frobify(&self) {
        println!("Frobifying a <{}>", self.name);
    }
}

// Yes, I know in the modern actual DOM an attribute is not a Node...but this makes for a
// nifty simple example
struct Attribute<'a>: NodeFields<'a> {
    name: ~str,
    value: ~str
}
impl<'a> Node<'a> for Attribute<'a> {
    fn frobify(&self) {
        // We know only elements have attributes as children, so we can downcast [3]
        let parent_element:&Element = downcast(parent.unwrap()).unwrap();
        println("Frobifying the attribute of {}: {}={}", parent_element.name, self.name, self.value);
    }
}

// No `unsafe` code was needed for any of the above [6]

```

## JDM's Servo Example

Here is [jdm's servo example](https://gist.github.com/jdm/9900569) translated into rust+this proposal:

```rust
struct NodeFields {
    parent: Rc<Fat<Node>>,
    first_child: Rc<Fat<Node>>
}
impl NodeFields {
    fn as_text_node<'a>(&'a self) -> &'a TextNode {
        assert!(/*sensible check*/);
        downcast(self)
    }
}
trait Node:NodeFields {
    fn as_element<'a>(&'a self) -> Option<&'a Element> {
        None
    }
}

struct TextNodeFields: NodeFields;
trait TextNode: Node {
    // ...
}
impl TextNode for TextNodeFields {
    // ...
}

struct ElementFields: NodeFields {
    attrs: Vec<(~str,~str)>
}
trait ElementStaticMethods:ElementFields {
    fn set_attribute(&mut self, key:&str, value:&str);
}
impl<T:Element> ElementStaticMethods for T {
    fn set_attribute(&mut self, key:&str, value:&str) {
        self.before_set_attr(key,value);
        // ...update attrs...
        self.after_set_attr(key,value);
    }
}
trait Element: ElementFields Node {
    fn before_set_attr(&mut self, key:&str, value:&str) {
        // Implementation goes here--jdm's gist does not give an implementation
        // but implies it exists (the method is not pure virtual)
    }
    fn after_set_attr(&mut self, key:&str, value:&str) {
        // Implementation goes here--jdm's gist does not give an implementation
        // but implies it exists (the method is not pure virtual)
    }
}

impl<T:Element> Node for T {
    fn as_element<'a>(&'a self) -> Option<&'a Element> {
        Some(self as &Element)
    }
}

struct HTMLImageElement: ElementFields;
impl Element for HTMLImageElement {
    fn before_set_attr(&mut self, key:&str, value:&str) {
        if key == "src" {
            //..remove cached image with url /value/...
        }
        // Should specify that this works...wouldn't in current rust, but would
        // it under the UFCS proposal?
        Element::before_set_attr(self,key, value);
    }
}

struct HTMLVideoElement: ElementFields {
    cross_origin:bool
}
impl Element for HTMLVideoElement {
    fn after_set_attr(&mut self, key:&str, value:&str) {
        if key == "crossOrigin" {
            self.cross_origin = (value == "true");
        }
        Element::after_set_attr(self,key,value);
    }
}

fn process_any_element(element:&Element) {
    //...
}

// Code outside of a function in jdm's example:
fn more_code() {
    // We assume we need videoElement to be a fat object for the trait Element, e.g.
    // because it would need to be stored in another element.
    let videoElement:Rc<Fat<Element, HTMLVideoElement>> = Rc::new(fat(/*...*/));
    process_any_element(&**videoElement);
    // We are cloning the Rc, not the node itself
    let node = videoElement.first_child.clone();
    match node.as_element() {
        Some(element) => { 
            // do things with element...
        }
        None => {
            let text = node.as_text_node();
            // ...
        }
    }
}
```

Some comments on this:

Note that for functions that take borrowed pointers as parameters, we still use fat pointers.  This is more flexible for the caller, as it allows the method to be called using both fat and thin pointers.  The cost of the virtual call will be roughly the same either way.  If the method needed a fat object, it would have to take a thin pointer to it as a parameter instead (e.g. `fn process_any_element(element:&Fat<Element>)`).  I would expect this would mostly appear methods that construct data structures that have thin pointers inside them.

The annoying part is having to create extra types that don't mean anything.  You don't just have `Element`, you have `ElementFields` and `ElementStaticMethods`, even though the second and third of these aren't really useful.  (`ElementStaticMethods` is here so that a call to `set_attribtue` is not itself virtual, but can make virtual calls to `before_set_attribute` and `after_set_attribute`.)  Choosing a different arrangement for struct inheritance could alleviate this--[traits with fields](https://github.com/mozilla/rust/issues/9912) may come out as a better complement here.  (_Possibly rewrite this RFC and choose that alternative instead?_)  With this example, at least the leaves of the inheritance tree (`HTMLImageElement` and `HTMLVideoElement`) can simply be structs and are expressed in a more straightforward way.

# Alternatives

The main alternative to this is [`virtual struct` and `virtual fn`](https://github.com/rust-lang/rfcs/pull/5).  Mere mention of this in the work week meeting notes generated a [lengthy email thread](http://thread.gmane.org/gmane.comp.lang.rust.devel/8878) about that idea.  The motivation for this RFC was to present an alternative to this.

After this, @bill-myers proposed a [completely different alternative](https://github.com/rust-lang/rfcs/pull/11) involving extending the notion of an `enum` to turn it into a syntax to describe an object hierarchy.  @nick29581 filed what he described as an [extension/variant](https://github.com/rust-lang/rfcs/pull/24) of this, which keeps the extension of enums, but then adds back `virtual` on top of it.

Some [brief thoughts on reddit](http://www.reddit.com/r/rust/comments/20b3dz/virtual_fn_is_a_bad_idea/cg1irqt) (with a link to some [discussion on irc](http://irclog.gr/#show/irc.mozilla.org/servo/76862)) by @eddyb talk about something very similar to the fat objects presented here.  Like this proposal, it allows for either thin or fat objects, but instead of choosing between thin and fat for each object, it would be chosen for each struct once at the struct's declaration.  This could happen either through a special `Vtable<Trait>` type (added explicitly to the struct), or via an attribute on the struct.  He says in the comment that he may be working on an RFC, so we may find more details about this soon.

There are multiple existing proposals floating around for how to implement single inheritance.  Alternatives to it include [letting traits specify fields](https://github.com/mozilla/rust/issues/9912), and [Coercible and HasPrefix traits](https://github.com/mozilla/rust/issues/9912#issuecomment-36073562).  Any of these would, in combination with the fat objects and downcast proposals in this RFC, fulfill the noted requirements.  The `HasPrefix`/`Coercible` trait proposal in particular may be quite appealing when combined with this--it would allow you to write syntactically the constraints on `downcast`, for instance (`fn downcast<Derived,Base:Coercible<Derived>>(b:Base) -> Derived`).  But for the purposes of this RFC I chose a syntactically simpler version instead because I didn't want to focus on it.

# Unresolved questions

Intrinsics are added to `std` but the exact path is not specified.

I'm sure there are other implementation concerns and edge case questions I haven't thought about.

This RFC has not addressed exactly how, in an overridden method, a parent method can be invoked from an overridden one.  The RFC for [unified function call syntax](https://github.com/rust-lang/rfcs/pull/4) could hopefully address this in a general way, for all traits rather than just structural inheritance.  (It would be good to be able to call the default impl from any trait when overriding it.)
