- Feature Name: duration_checked_sub
- Start Date: 2016-06-04
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

This RFC adds `checked_sub()` already known from various primitive types to the
`Duration` *struct*.

# Motivation
[motivation]: #motivation

Generally this helps when subtracting `Duration`s which can be the case quite
often.

One abstract example would be executing a specific piece of code repeatedly
after a constant amount of time.

Specific examples would be a network service or a rendering process emitting a
constant amount of frames per second.

Example code would be as follows:

```rust

// This function is called repeatedly
fn render() {
    // 10ms delay results in 100 frames per second
    let wait_time = Duration::from_millis(10);

    // `Instant` for elapsed time
    let start = Instant::now();

    // execute code here
    render_and_output_frame();

    // there are no negative `Duration`s so this does nothing if the elapsed
    // time is longer than the defined `wait_time`
    start.elapsed().checked_sub(wait_time).and_then(std::thread::sleep);
}
```

# Detailed design
[design]: #detailed-design

The detailed design would be exactly as the current `sub()` method, just
returning an `Option<Duration>` and passing possible `None` values from the
underlying primitive types:

```rust
impl Duration {
    fn checked_sub(self, rhs: Duration) -> Duration {
        if let Some(mut secs) = self.secs.checked_sub(rhs.secs) {
            let nanos = if self.nanos >= rhs.nanos {
                self.nanos - rhs.nanos
            } else {
                if let Some(secs) = secs.checked_sub(1) {
                    self.nanos + NANOS_PER_SEC - rhs.nanos
                }
                else {
                    return None;
                }
            };
            debug_assert!(nanos < NANOS_PER_SEC);
            Duration { secs: secs, nanos: nanos }
        }
        else {
            None
        }
    }
}
```

# Drawbacks
[drawbacks]: #drawbacks

This proposal adds another `checked_*` method to *libstd*.
One could ask why no `CheckedSub` trait is used as is done with `Add`.

# Alternatives
[alternatives]: #alternatives

The alternatives are simply not doing this and forcing the programmer to code
the check on their behalf.

# Unresolved questions
[unresolved]: #unresolved-questions

`None`.

