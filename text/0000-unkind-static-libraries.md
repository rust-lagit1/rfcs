- Feature Name: more_link_kinds
- Start Date: 2016-02-05
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary


Adds a new `kind=resource` that is used to link resource files.

# Motivation
[motivation]: #motivation

Resource files are a necessary thing for any serious application on Windows. They allow you to specify icons, manifests, version information, all sorts of stuff. However, Rust currently lacks a first class way to link to resource files, instead requiring that the user find the necessary tools themselves to compile the resource file, and then rename it to look like a library so it can trick rustc into passing it to the linker via `kind=dylib`. By adding first class support for resource files, rustc would be responsible for finding the necessary tools to compile a `foo.rc` into a `foo.res` and then passing that to the linker.

# Detailed design
[design]: #detailed-design

`kind=resource` can be applied the same way as any of the other `kind`s, whether via flags passed to cargo via build scripts, flags passed to rustc via the command line, or `#[link]` attributes. rustc will use the same mechanism as it does with `kind=static` to search for the resource file. It will then locate the resource compiler and compile the resource file like so:

* MSVC: `rc foo.rc` which will create a `foo.res`.
* MinGW: `windres -i foo.rc -o foo.res` will create a `foo.res`.

The compiled `foo.res` will then be saved somehow so that when rustc does finally invoke the linker, it will pass that `foo.res` to the linker.

In the future it may be possible to have rustc use a library to compile resource files, eliminating the need for external tools. I know at least one person was working on such a library in Rust.

Due to resource files depending on other files around them like icons and manifests, the resource file has to be compiled on the spot, it cannot be stored to be later compiled.

Due to compiled resource files not working when they are passed to the linker when inside a library, they will have to be separate files that are passed to the linker separately.

## dllimport and dllexport

Symbols from a resource file are assumed to be static symbols so `dllimport` will not be applied.

# Drawbacks
[drawbacks]: #drawbacks

* It adds another `kind` that has to be supported and tested.

# Alternatives
[alternatives]: #alternatives

* Instead of using a new `kind`, this could be done via a new attribute, or via a special rustc flag, along with support in cargo build scripts somehow as well.
* Maintain the status quo where users have to have their build scripts locate resource compilers, compile the resource file, rename it to look like a library, and trick rustc into passing it to the linker via `kind=dylib`. There are multiple third party solutions for this, and all of them actually did this workaround wrong (until I corrected them).

# Unresolved questions
[unresolved]: #unresolved-questions

* Do we just save the path to the compiled `.res` or do we actually store it inside the `.rlib` and then later extract it and pass it to the linker?
* The name of the `kind`. Please bikeshed vigorously.
* Does it make sense to be able to have an extern block for a resource file with symbols inside of it? If not, is `#[link]` even the right thing, or is there a better syntax we could use?
* There are a variety of flags that can be passed to `rc.exe`, most of which I can't find an equivalent to in `windres`. Are any of these flags necessary? Would we need to provide a way to specify these flags?
