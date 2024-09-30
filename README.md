# linker search path of proc_macro deps are not propagated?

This is a bug report for [rules_rust][].

## Reproduction Requirements

I could trigger this on macOS only. Install:

```shell
$ brew install bazelisk postgresql@14 pkg-config
```

## What happens here

`rust_decimal` is a regular library crate for decimal numbers. With the `'db-diesel2-postgres'` feature,
it pulls in the `pq-sys` dependency, that in its `build.rs` potentially finds libpq using `pkg-config`:

```
cargo:rustc-link-search=native=/opt/homebrew/lib/postgresql@14
cargo:rustc-link-lib=pq
```

(To make `pkg-config` work with the homebrew setup, we are modifying some envvars, see `.bazelrc`)

In this demo repository, the target `//:good` depends on `rust_decimal`, and bazel can compile and link it.

The `rust_decimal_macros` is a proc-macro crate, that provides (surprise!) macros for the above crate.
It depends on `rust_decimal`, therefore it also pulls in `pq-sys` transitively.

In this demo repository, the target `//:bad` depends on `rust_decimal_macros`, and bazel cannot compile it:

```
$ bazel build bad
[...]
ld: library 'pq' not found
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

## Suspicion

`aquery` shows that `//:good` inherits a linker search path from `pq-sys`:

```
$ bazel aquery good | grep pq
[...]
bazel-out/darwin_arm64-fastbuild/bin/external/rules_rust~~crate~crates__pq-sys-0.6.3/_bs.linksearchpaths \
```

but `//bad:` does not. Therefore the suspicion, that proc macro crates do not pass forward their link search path files?

Interestingly, even if `bad` also depends on `rust_decimal` or even `pq-sys`, it doesn't link.

[rules_rust]: https://bazelbuild.github.io/rules_rust/
