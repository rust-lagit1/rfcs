- Feature Name: Target Version
- Start Date: 2015-06-03
- RFC PR: 
- Rust Issue: 

# Summary

This RFC proposes a small number of extensions to improve the user
experience around different rust versions, while at the same time
giving Rust developers more freedom to make changes without breaking
anyones code.

Namely the following items:

1. Add a `--target-version=`*<version string>* command line argument to 
   rustc. This will be used for deprecation checking and for selecting 
   code paths in the compiler.
2. Add an optional `rust = "..."` package attribute to Cargo.toml, 
   which `cargo new` pre-fills with the current rust version.
3. Add a `removed_at="..."` item to `#[deprecated]` attributes that 
   allows making API items unavailable starting from certain target 
   versions.
4. Add a number of warnings to steer users in the direction of using 
   the most recent Rust version that makes sense to them, while making 
   it easy for library writers to support a wide range of Rust versions
5. (optional) add a `legacy="..."` item to `#[deprecated]` attributes
   that allows grouping API items under a legacy flag that is already 
   defined in RFC #1122 (see below)

## Background

A good number of policies and mechanisms around versioning regarding
the Rust language and APIs have already been submitted, some accepted.
As a background, in no particular order:

* [#0572 Feature gates](https://github.com/rust-lang/rfcs/blob/master/text/0572-rustc-attribute.md)
* [#1105 API Evolution](https://github.com/rust-lang/rfcs/blob/master/text/1105-api-evolution.md)
* [#1122 Language SemVer](https://github.com/rust-lang/rfcs/blob/master/text/1122-language-semver.md)
* [#1150 Rename Attribute](https://github.com/rust-lang/pull/1150)

In addition, there has been an ongoing 
[discussion on internals](https://internals.rust-lang.org/t/thoughts-on-aggressive-deprecation-in-libstd/2176/55) 
about how we are going to evolve the standard library, which this
proposal is somewhat based on.

Finally, the recent discussion on the first breaking change 
([RFC PR #1156 Adjust default object bounds](https://github.com/rust-lang/rfcs/pull/1156))
has made it clear that we need a more flexible way of dealing with
(breaking) changes.

The current setup allows the `std` API to declare `#[unstable]` and
`#[deprecated]` flags, whereas users can opt-in to unstable features
with `#![feature]` flags that are customarily added to the crate root.
On usage of deprecated APIs, a warning is shown unless suppressed.
`cargo` does this for dependencies by default by calling `rustc` with
the `-Awarnings` argument.

# Motivation

## 1. Language / `std` Evolution

The following motivates items 1 and 2 (and to a lesser extent 4)

With the current setup, we can already evolve the language and APIs,
albeit in a very limited way. For example, it is virtually impossible
to actually remove an API, because that would break code. Even minor
breaking changes (as [RFC PR #1156](https://github.com/rust-lang/rfcs/pull/1156) 
cited above) generate huge discussion, because they call the general
stability of the language and environment into question.

The problem with opt-out, as defined by 
[RFC #1122](https://github.com/rust-lang/rfcs/blob/master/text/1122-language-semver.md)
is that code which previously compiled without problems stops working
on a Rust upgrade, and requires manual intervention to get working 
again.

This has the long-term potential of fragmenting the Rust ecosystem into
crates/projects using certain Rust versions and thus should be avoided
at some (but not all) cost.

Note that a similar problem exists with deprecation as defined: People
may get used to deprecation warnings or just turn them off until their
build breaks. Worse, the current setup creates a mirror world for
libraries, in which deprecation doesn't exist!

## 2. User Experience

The following motivates items 2, 3 and 4.

Currently, there is no way to make an API item unavailable via 
deprecation. This means the API will only ever expand, with a lot of
churn (case in point, the Java language as of Version 8 lists 462
deprecated constructs, including methods, constants and complete 
classes, in relation to 4241 classes, this makes for about 10% of
deprecated API surface. Note that this is but a rough estimate). To 
better lead users to the right APIs and to allow for more effective 
language/API evolution, this proposal adds the `removed_at="<version>"`
item to the `#[deprecated]` attribute.

This allows us to effectively remove an API item from a certain target
version while avoiding breaking code written for older target versions.

Also rustc can emit better error messages than it could were the API
items actually removed. In the best case, the error messages can steer
the user to a working replacement.

We want to avoid users setting `#[allow(deprecate)]` on their code to
get rid of the warnings. On the other hand, there have been instances
of failing builds because code was marked with `#![deny(warnings)]`,
namely `compiletest.rs` and all crates using it. This shows that the
current system has room for improvement.

We want to avoid failing builds because of wrong or missing target
version definition. Therefor supplying useful defaults is of the 
essence.

The documentation can be optionally reduced to items relating to the
current target version (e.g. by a switch), or deprecated items 
relegated to a separate space, to reduce clutter and possible user
confusion.

# Detailed design

Cargo parses the additional `rust = "..."` package attribute. The usual 
rules for version parsing apply. If no `rust` attribute is supplied, it 
defaults to `*`.

Cargo should also supply the current Rust version (which can be either
supplied by calling `rustc -V` or by linking to a rust library defining
a version object) on `cargo new`. Cargo supplies the given target 
version to `rustc` via the `--target-version` command line argument.

Cargo *may* also warn on `cargo package` if no `rust` version was 
supplied. A few versions in the future, Cargo could also warn of 
missing version attributes on build or other actions, at least if the 
crate is a library.

[crates.io](https://crates.io) *could* require a version attribute on 
upload and display the required rust version on the site.

`rustc` needs to accept the `--target-version <version>` command line 
argument. If no argument is supplied, `rustc` defaults to its own 
version. The same version syntax as Cargo applies:

* `*` effectively means *any version*. For API items, it means
  deprecation checking is disabled. For language changes, it means 
  using the `1.0.0` code paths (for now, we may opt to change this in 
  the future), because anything else would break all current code.
* `1.x` or e.g. `>=1.2.0` sets the target version to the minor version. 
  Deprecation checking and language code path selection occur relative 
  to the lowest given version. This might also affect stability 
  handling, though this RFC doesn't specify this as of yet.
* `1.0 - <2.0` as above, the *lowest* supplied version has to be
  assumed for code path selection. However, deprecation checking should
  assume the *highest* supplied version, if any.

If the target version is *higher* than the current `rustc` version, 
`rustc` should show a warning to suggest that it may need to be updated 
in order to compile the crate and then try to compile the crate on a
best-effort basis.

Optionally, we can define a `future deprecation` lint set to `Allow` by 
default to allow people being proactive about items that are going to
be deprecated.

`rustc` should resolve the `#[feature]` flags against the upper bound 
of the specified target version instead the current version, but 
default to the current version if no target version is specified or the 
specified version has no upper bound.

While `rustdoc` already parses the deprecation flags, it should in 
addition relegate items removed in the current version to a separate 
area below the other documentation and optically mark them as removed. 
We should not completely remove them, because that would confuse users 
who see the API items in code written for older target versions.

# Optional Extension: Legacy flags

The `#[deprecated]` attribute could get an optional `legacy="xy"`
entry, which could effectively group a set of APIs under the given 
name. Users can then declare the `#[legacy]` flag as defined in 
[RFC #1122](https://github.com/rust-lang/rfcs/blob/master/text/1122-language-semver.md)
to specifically allow usage of the grouped APIs, thus selectively
removing the deprecation warning.

This would create a nice feature parity between language code paths and
`std` API deprecation checking. Also it would lessen the pain for users
who want to upgrade their systems one feature at a time and can use the
legacy flags to effectively manage their usage of deprecated items.

# Drawbacks

By requiring full backwards-compatibility, we will never be able to 
actually remove stuff from the APIs, which will probably lead to some 
bloat. However, the cost of maintaining the outdated APIs is far 
outweighted by the benefits. Case in point: Other successful languages 
have lived with this for multiple decades, so it appears the tradeoff 
has seen some confirmation already. 

Cargo and `rustc` need some code to manage the additional rules. I
estimate the effort to be reasonably low. For *compiler changes* 
however, unless it's a genuine bug and unless there could be programs 
relying on the old behaviours, both the old and new code paths have to
be maintained in the compiler, which is the biggest cost of 
implementing this RFC.

# Alternatives

* An earlier version of this proposal suggested using a crate attribute
  instead of a cargo package attribute and a compiler option, to also 
  allow cargo-less use cases without manual interaction. However it was 
  determined that those cases usually target the current Rust version 
  anyway, and the current proposal allows us to default to the current 
  version for rustc, while the earlier proposal would have defaulted to 
  1.0.0 by necessity of not breaking existing code

* It was suggested that opt-in and opt-out (e.g. by `#[legacy(..)]`)
  could be sufficient to work around any breaking code on API or 
  language changes. The big problem here is that this relies on the 
  user being able to change their dependencies, which may not be 
  possible for legal, organizational or other reasons. In contrast, a 
  defined target version doesn't ever need to change

  Depending on the specific case, it may be useful to allow a 
  combination of `#![legacy(..)]`, `#![feature(..)]` and the target 
  version where each Rust version can declare the currently active 
  feature set and permit or forbid use of the opt-in/out flags

* Follow a more agressive strategy that actually removes stuff from the 
  API. This would make it easier for the libstd creators at some cost 
  for library and application writers, as they are required to keep up 
  to date or face breakage. The risk of breaking existing code makes 
  this strategy very unattractive

* Hide deprecated items in the docs: This could be done either by 
  putting them into a linked extra page or by adding a "show 
  deprecated" checkbox that may be default be checked or not, depending 
  on who you ask. This will however confuse people, who see the 
  deprecated APIs in some code, but cannot find them in the docs 
  anymore 

* Allow to distinguish "soft" and "hard" deprecation, so that an API 
  can be marked as "soft" deprecated to dissuade new uses before hard 
  deprecation is decided. Allowing people to specify deprecation in 
  future version appears to have much of the same benefits without 
  requiring a new attribute key

# Unresolved questions

* The names for the cargo package attribute and the rustc compiler 
  option are still subject to bikeshedding (however, discussion has 
  stalled, suggesting the current names are good enough).

* How do we determine if something is a genuine bug (and should be 
  changed retroactively)?

* If we agree that something needs to be changed retroactively (i.e. in 
  older versions), do we also release the old versions anew? Which 
  ones? Should we nominate LTS versions? Who would maintain them?

* Is *forward-compatibility* sufficiently handled? Seeing that e.g. 
  adding an item to a trait could break code using that trait, changing 
  a trait would require both versions being interoperable, which could 
  be impossible in the general case. This would needed to be handled by 
  finding a new name for the trait or supplying a default 
  implementation.
