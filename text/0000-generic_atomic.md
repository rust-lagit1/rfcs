- Feature Name: generic_atomic
- Start Date: 21-02-2016
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes adding a generic `Atomic<T>` type which can accept any `T: Copy`. The actual set of types that are accepted is restricted based on what the target supports: if `T` is too big then compilation will fail with an error indicating that atomics of the required size are not supported. The actual atomic API is the same as the existing one, so there is nothing new there. However this will require compiler support for an `AtomicWrapper` type which is used internally.

```rust
#[atomic_wrapper]
struct AtomicWrapper<T: Copy>(T);

pub struct Atomic<T: Copy> {
    val: UnsafeCell<AtomicWrapper<T>>
}

impl<T: Copy> Atomic<T> {
    pub const fn new(val: T) -> Atomic<T>;
    pub fn load(&self, order: Ordering) -> T;
    pub fn store(&self, val: T, order: Ordering);
    pub fn exchange(&self, val: T, order: Ordering) -> T;
    pub fn compare_exchange(self, current: T, new: T, success: Ordering, failure: Ordering) -> T;
    pub fn compare_exchange_weak(self, current: T, new: T, success: Ordering, failure: Ordering) -> (T, bool);
}

impl Atomic<bool> {
    pub fn fetch_and(&self, val: bool, order: Ordering) -> bool;
    pub fn fetch_nand(&self, val: bool, order: Ordering) -> bool;
    pub fn fetch_or(&self, val: bool, order: Ordering) -> bool;
    pub fn fetch_xor(&self, val: bool, order: Ordering) -> bool;
}

impl Atomic<i8> { // And other integer types. i64/u64 only if supported by target.
    pub fn fetch_add(&self, val: i8, order: Ordering) -> i8;
    pub fn fetch_sub(&self, val: i8, order: Ordering) -> i8;
    pub fn fetch_and(&self, val: i8, order: Ordering) -> i8;
    pub fn fetch_or(&self, val: i8, order: Ordering) -> i8;
    pub fn fetch_xor(&self, val: i8, order: Ordering) -> i8;
}
```

# Motivation
[motivation]: #motivation

Many lock-free algorithms require a two-value `compare_exchange`, which is effectively twice the size of a `usize`. This would be implemented by atomically swapping a struct containing two members.

Another use case is to support Linux's futex API. This API is based on atomic `i32` variables, which currently aren't available on x86_64 because `AtomicIsize` is 64-bit.

Finally, many people coming from C++ will expect a generic atomic type like `std::atomic<T>`.

# Detailed design
[design]: #detailed-design

## `Atomic<T>`

This is fairly straightforward: `load`, `store`, `exchange`, `compare_exchange` and `compare_exchange_weak` are implemented for all atomic types. `bool` and integer types have additional `fetch_*` functions, which match those in `AtomicBool`, `AtomicIsize` and `AtomicUsize`.

The only complication is that `compare_exchange` does a bitwise comparison, which may fail due to differences in the padding bytes of `T`. This is solved with an intrinsic that explicitly clears the padding bytes of a value using a mask. We have to do this ourselves rather than relying on the user because Rust struct layouts are imlpementation-defined.

## `AtomicWrapper<T>`

This is an implementation detail which requires special compiler support. It has two functions:

- Rounds the size and alignment of `T` up to the next power of two.

- Gives an error message if `T` is too big for the current target's atomic operations.

Any extra padding added by `AtomicWrapper<T>` will be cleared before it is used in an atomic operation.

## Target support

One problem is that it is hard for a user to determine if a certain type `T` can be placed inside an `Atomic<T>`. After a quick survey of the LLVM and Clang code, architectures can be classified into 3 categories:

- The architecture does not support any form of atomics (mainly microcontroller architectures).
- The architecture supports all atomic operations for integers from i8 to iN (where N is the architecture word/pointer size).
- The architecture supports all atomic operations for integers from i8 to i(N*2).

A new target cfg is added: `target_has_atomic`. It will have multiple values, one for each atomic size supported by the target. For example:

```rust
#[cfg(target_has_atomic = "128")]
static ATOMIC: Atomic<(u64, u64)> = Atomic::new((0, 0));
#[cfg(not(target_has_atomic = "128"))]
static ATOMIC: Mutex<(u64, u64)> = Mutex::new((0, 0));

#[cfg(target_has_atomic = "64")]
static COUNTER: Atomic<u64> = Atomic::new(0);
#[cfg(not(target_has_atomic = "64"))]
static COUTNER: Atomic<u32> = Atomic::new(0);
```

Note that it is not necessary for an architecture to natively support atomic operations for all sizes (`i8`, `i16`, etc) as long as it is able to perform a `compare_exchange` operation with a larger size. All smaller operations can be emulated using that. For example a byte atomic can be emulated by using a `compare_exchange` loop that only modifies a single byte of the value. This is actually how LLVM implements byte-level atomics on MIPS, which only supports word-sized atomics native. Note that the out-of-bounds read is fine here because atomics are aligned and will never cross a page boundary. Since this transformation is performed transparently by LLVM, we do not need to do any extra work to support this.

# Drawbacks
[drawbacks]: #drawbacks

`AtomicWrapper` relies on compiler magic to work.

Having certain atomic types get enabled/disable based on the target isn't very nice, but it's unavoidable.

`Atomic<bool>` will have a size of 1, unlike `AtomicBool` which uses a `usize` internally. This may cause confusion.

# Alternatives
[alternatives]: #alternatives

Rather than generating a compiler error, unsupported atomic types could be translated into calls to external functions in `compiler-rt`, like C++'s `std::atomic<T>`. However these functions use locks to implement atomics, which makes them unsuitable for some situations like communicating with a signal handler.

Several other designs have been suggested [here](https://internals.rust-lang.org/t/pre-rfc-extended-atomic-types/3068).

# Unresolved questions
[unresolved]: #unresolved-questions

How should we deal with architectures that don't support native atomic operations (for example, ARMv5)? Should we disallow atomics entirely? Should we silently fall back to lock-based implementations?
