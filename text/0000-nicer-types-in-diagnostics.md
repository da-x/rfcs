- Feature Name: Nicer Types In Diagnostics
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

When types are printed in diagnostics, each type's fully qualified type incurs a cognitive load on the reader. For example, printing the type
`std::vec::Vec<std::vec::Vec<std::vec::Vec<u16>>>` could have been shortened to `Vec<Vec<Vec<u16>>>`. This proposal seeks to remedy the situation, by omitted the type's path when possible, however it also seeks to prevent ambiguity.

# Motivation
[motivation]: #motivation

The expected outcome is that it will be easier to read type errors, but without losing the ability to pin-point the exact problem to the user.
Currently, a fully qualified type path precisely indicates a type, and we need to make sure that stripping the path does not create ambiguity.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

First, for let's define how each non-qualified type falls into either of three categories:

* Types that are imported by default from prelude.
* Types that are imported in the scope in which the type errors occurs, and are not prelude types.
* All other types that don't fall into the first two categories.

In the most reduced from of types printing setting named `minimal`, we differentiate between the three categories.

* For prelude types, as long as the type is not occluded by an import with the same name, we omitted the path entirely.
* For imports that are available in the current scope, we also omit the type path, but introduce a `via imports` line
  to the type error (see below), illustrative of the `use` statement that imports the type.
* For all other types, as long as the type is not occluded by an import with the same name, we omit the path, but also introduce a similar `reachable by` line to the error message.

Consider the following example:

```rust
fn test3(v: &Vec<Vec<Vec<u16>>>) {}

fn main() {
    use std::collections::HashMap;

    let ip: std::net::IpAddr = "0.0.0.0".parse().unwrap();
    test3(vec![vec![(String::from("x"), ip, HashMap::new())]]);
}
```

The output will be:

```rust
7 |     test3(vec![vec![(String::from("x"), ip, HashMap::new())]]);
  |           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected reference, found struct `Vec`
  |
  = note: expected type `&Vec<Vec<Vec<u16>>>`
             found type `Vec<Vec<(String, IpAddr, HashMap<_, _>)>>`
            via imports `use std::collections::HashMap;`
           reachable by `use std::net::IpAddr;`
```

Instead of the `uniform`, default setting thus far:

```rust
 --> src/case2.rs:7:11
  |
7 |     test3(vec![vec![(String::from("x"), ip, HashMap::new())]]);
  |           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected reference, found struct `std::vec::Vec`
  |
  = note: expected type `&std::vec::Vec<std::vec::Vec<std::vec::Vec<u16>>>`
             found type `std::vec::Vec<std::vec::Vec<(std::string::String, std::net::IpAddr, std::collections::HashMap<_, _>)>>`
```

For the transition period, there is also a `by-import` setting that limits reduction of type paths for imports, preventing the the `reachable by` segment.

```rust
7 |     test3(vec![vec![(String::from("x"), ip, HashMap::new())]]);
  |           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected reference, found struct `Vec`
  |
  = note: expected type `&Vec<Vec<Vec<u16>>>`
             found type `Vec<Vec<(String, std::net::IpAddr, HashMap<_, _>)>>`
            via imports `use std::collections::HashMap;`
```

Also for the transition period, the three settings can be controlled via a crate-level `type_diagnostic` feature. For example:

```rust
#![feature(type_diagnostic)]
#![type_diagnostic = "minimal"]
```

[More examples in a very early draft rustc implementation of the RFC](https://github.com/da-x/rust/commit/a67af3dccb882d6858d1be78469db56eabd5b533).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The feature will be implemented via the following steps:

* Produce an `imports` map from `librustc_resolve` back to `librustc`.
* The imports will be a map from the module concept of `resolve` to the `DefId`
  that are defined in that scope, with their respective `Ident`s.
* The parent scope will be included and walkable from each `HirId` in
  `librustc/ty/print/pretty.rs`, so gathering up the `uses` in scope will be
  efficient.
* Code paths that emit errors will need to pass the `HirId` of the relevant
  scope, which will be stored in TLS for `pretty.rs` to catch.

[Very early implementation draft in rustc](https://github.com/da-x/rust/commit/1856989cdf2ba2fb956fc06a1a919f099e94dc7f).

(TBD: corner cases)

# Drawbacks
[drawbacks]: #drawbacks

We may choose not to do this because people are already too accustomed to
groking the current and long uniform paths.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

One alternative is that instead of `by-import` line, we quote the actual span
of the `use` statement that brought the type. However this may be hard to read,
and the `use` lines may be too numerous. Also, this does not apply to
`reachable via`.

The impact of not doing this is a newcomer's roadblock issue, as was already
realized in previous discussions.

# Prior art
[prior-art]: #prior-art

(TBD: needs research, am not aware of other languages doing something like this)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How those prints should be formatted exactly, to make them most readable?
- How do we modify the code for it? Is TLS an `okay` solution or should we do a
  fuller rewrite of the pretty-printing engine?
- Can or should we rely on `Span`s instead of `HirId`?
- Should the default for the setting be kept `uniform`, should we change the
  default, or get rid of the setting altogether?

# Future possibilities
[future-possibilities]: #future-possibilities

From an implementation standpoint. The `import_map` can be used by analysis
tools for cross-crate important analysis. Say, for determining if there dead
code in a set of modules comprising a namespace.

Also, tools such as `rustfix` may use the extra printed lines to augment the
`use` pseudo-section of a module.
