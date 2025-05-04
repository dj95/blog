---
title: "Learnings from developing a zellij plugin"
date: 2024-01-01T12:05:02+02:00
draft: false
tags: ["zellij", "rust", "tdd", "benchmarks"]
---

Zellij is a terminal multiplexer written in Rust, that aims to provide an intuitive and easy to extend working environment.
With version 0.37.0 Zellij introduced a plugin system, that allows users to write plugins with WebAssembly.
Additionally the plugin system provides a rust crate with an API to interact with the Zellij core.

Since zellij to the date of writing this article has not provided customizations for the status bar, I decided to write one that is easily customizable by writing KDL configuration into layout files - [zjstatus](https://github.com/dj95/zjstatus).

While writing, extending and refactoring the plugin, I learned a lot about the plugin system, WebAssembly and Rust in general.
Especially about performance impact in Rust when memory management is not implemented correctly.
In this article I want to share some of these insights.

In order to follow the article, please check out the official [rust-plugin-example](https://github.com/zellij-org/rust-plugin-example), which I used as a starting point.

## Initial setup

This section describes some actions I took to set up the development environment for simplifying the development process.

### Development dependencies

Since I'm a big fan of nix and nixOS, I started by creating a `shell.nix` file, that provides all the dependencies needed for development.
It contains a pinned version of the nixpkgs repository, that holds build instructions for used dependencies and their respective versions.

```nix
{ pkgs ? import <nixpkgs> {} }:
with pkgs;
let
  pinnedPkgs = fetchFromGitHub {
    owner = "NixOS";
    repo = "nixpkgs";
    rev = "a63a64b593dcf2fe05f7c5d666eb395950f36bc9";
    sha256 = "sha256-+ZoAny3ZxLcfMaUoLVgL9Ywb/57wP+EtsdNGuXUJrwg=";
  };

  pkgs = import pinnedPkgs {};

  inherit (lib) optional optionals;
  inherit (darwin.apple_sdk.frameworks) Cocoa CoreGraphics Foundation IOKit Kernel OpenGL Security;
in
  pkgs.mkShell rec {
    buildInputs = with pkgs; [
      clang
      # Replace llvmPackages with llvmPackages_X, where X is the latest LLVM version (at the time of writing, 16)
      llvmPackages.bintools
      rustup
      libiconv
      watchexec
    ];
    RUSTC_VERSION = pkgs.lib.readFile ./rust-toolchain;
    # https://github.com/rust-lang/rust-bindgen#environment-variables
    LIBCLANG_PATH = pkgs.lib.makeLibraryPath [ pkgs.llvmPackages_latest.libclang.lib ];
    shellHook = ''
      export PATH=$PATH:''${CARGO_HOME:-~/.cargo}/bin
      export PATH=$PATH:''${RUSTUP_HOME:-~/.rustup}/toolchains/$RUSTC_VERSION-x86_64-unknown-linux-gnu/bin/
      '';
    # Add precompiled library to rustc search path
    RUSTFLAGS = (builtins.map (a: ''-L ${a}/lib'') [
      # add libraries here (e.g. pkgs.libvmi)
    ]);
    # Add glibc, clang, glib and other headers to bindgen search path
    BINDGEN_EXTRA_CLANG_ARGS = 
    # Includes with normal include path
    (builtins.map (a: ''-I"${a}/include"'') [
      # add dev libraries here (e.g. pkgs.libvmi.dev)
      pkgs.libiconv
    ])
    # Includes with special directory paths
    ++ [
      ''-I"${pkgs.llvmPackages_latest.libclang.lib}/lib/clang/${pkgs.llvmPackages_latest.libclang.version}/include"''
      ''-I"${pkgs.libiconv.out}/lib/"''
    ];

  }
```

It will install `clang`, `llvm`, `rustup`, `libiconv` and `watchexec` into a nix shell, additionally to configure build dependencies for the rust compiler.
To automatically enable usage of these dependencies, I added a `.envrc` to utilze [direnv](https://direnv.net) for automatically loading the nix shell when entering the project directory.

```bash
use nix
```

Later on I switched to a `flake.nix` file to provide a convenient, reproducible and declarative way to build the project.

### Task runner

Even if most of the tasks while developing can be done by directly running `cargo` commands, I start by setting up a `justfile`, that is used by [just](https://github.com/casey/just), an alternative to `make`.
Then `justfile` will provide a unified interface to run all the commands needed for development and also lists them conveniently.

```make
build:
  cargo build

run: build
  zellij -l ./plugin-dev-workspace.kdl -s zjstatus-dev

test:
  cargo component test -- --nocapture

lint:
  cargo clippy --all-targets -- -D warnings
  cargo audit

release version:
  cargo set-version {{version}}
  direnv exec . cargo build --release
  git commit -am "chore: bump version to v{{version}}"
  git tag -m "v{{version}}" v{{version}}
  git cliff --current
```

Commands can be run by calling `just <command>`, e.g. `just run`.
The syntax for releases allows to bump the version and create a git tag with a single command: `just release 0.1.0`.

### Splitting code to binary and library

As we want to write tests and benchmarks for the code, we need to split the entrypoint of the plugin from the library code.
All parts that we want to test and benchmark need to be part of the library crate.
To split the binary from the library, we need to modify the file structure in the following way:

```bash
src/main.rs -> src/bin/plugin-name.rs # move
src/lib.rs # create
```

The current main file will be moved to `src/bin/plugin.rs` and the `src/lib.rs` file will be created.
`src/lib.rs` will contain the library code, that contains logic for the plugin.
In addition to the directory structure, we need to modify the `Cargo.toml` file to reflect the changes:

```toml
[[bin]]
name = "plugin-name"
bench = false

[lib]
bench = false
```

This will prevent cargo from directly running library or binary code, when only benchmarks should be run.
It is important to configure the project like that because zellij's API with `register_plugin` will try to connect to the running zellij instance, which is not present when running benchmarks.
Since all logic related code is now located in the library, we need to import it in the binary or benchmarks with `use plugin_name::*;`.

## Unit testing

For ensuring the correctness of plugin logic and to validate, that the logic works as expected when refactoring the code, I am a big fan of unit testing.
It not only helps to ensure correctness, but also provides a way to document the code and its usage.
In addition it helps to develop new features in a test driven way, such that the test is written before the actual implementation.

Unit tests in zellij plugins can be written by adding a `#[cfg(test)]` attribute to the test module and writing tests with the `#[test]` attribute - like in any other rust project.
You just need to make sure, that code that interacts with zellij's API is not executed when running tests, as it will fail otherwise.
This can be done by using the `#[cfg(not(test))]` attribute on the code that should not be executed when running tests.

Another tricky part is to run the unit tests, as they need to be run within a WebAssembly runtime (like `wasmtime`).
To simplify running the tests, I added [cargo-component](https://github.com/bytecodealliance/cargo-component) to the project dependencies, which runs code with `wasmtime` after it has been compiled.
A shortcut for running the tests was also added to the `justfile` before.

Here's an example of a unit test, that verifies the correctness of the `parse_color` function:

```rust
#[cfg(test)]
mod test {
    use super::*;

    #[test]
    fn test_parse_color() {
        let result = parse_color("#010203");
        let expected = RgbColor(1, 2, 3);
        assert_eq!(result, Some(expected.into()));

        let result = parse_color("255");
        let expected = Ansi256Color(255);
        assert_eq!(result, Some(expected.into()));

        let result = parse_color("365");
        assert_eq!(result, None);

        let result = parse_color("#365");
        assert_eq!(result, None);
    }
}
```

## Benchmarks

Benchmarks can be used to check for performance regressions when refactoring code.
They can also be used to measure the performance impact of new features.
Therefore it is important to store results of the benchmarks, to compare them with future runs.

Normally, I like to use [divan](https://github.com/nvzqz/divan) for writing benchmarks, but it does not support WebAssembly yet.
Therefore I switched to [criterion](https://github.com/bheisler/criterion.rs) since it supports WebAssembly and provides some convenient features for storing and comparing benchmark results.

To start with benchmarks, first add `criterion` and the benchmark targets to the `Cargo.toml` file:

```toml
[dev-dependencies]
criterion = { version = "0.5.1", default-features = false, features = ["html_reports"] }

[[bench]]
name = "benches"
harness = false
```

Benchmark targets must be named like the benchmark files in the benches directory, e.g. `benches/benches.rs`.

Follow the [criterion documentation](https://bheisler.github.io/criterion.rs/book/criterion_rs.html) for writing benchmarks.
Here's a quick example:

```rust
use criterion::{criterion_group, criterion_main, Criterion};

use zjstatus::render::formatted_parts_from_string_cached;

fn bench_formattedparts_from_format_string_cached(c: &mut Criterion) {
    c.bench_function("formatted_parts_from_string_cached", |b| {
        b.iter(|| {
            formatted_parts_from_string_cached(
                "#[fg=#9399B2,bg=#181825,bold,italic] {index} {name} [] #[fg=#9399B2,bg=#181825,bold,italic] {index} {name} [] ",
            )
        })
    });
}

fn criterion_benchmark(c: &mut Criterion) {
    bench_formattedparts_from_format_string_cached(c);
}

criterion_group!(benches, criterion_benchmark);
criterion_main!(benches);
```

Similar to unit tests we need to make sure that code which interacts with zellij's API is not executed when running benchmarks.
Therefore we need to add a new feature to the `Cargo.toml` and add the `#[cfg(not(feature = "bench"))]` attribute to the code that should not be executed when running benchmarks.

```toml
[features]
bench = []
```

Since criterion needs to store benchmark results in the `target` directory, we cannot run benchmarks with `cargo component bench`.
`cargo component bench` will not only try to run the plugin wasm binary outside of zellij, but also runs the benchmarks in `wasmtime` without granting permissions to write to the `target` directory.
Therefore we need to compile benchmarks with `cargo bench --target wasm32-wasi --benches --no-run --feature=benches`.
After compilation it will print the path to the compiled binaries.
These binaries then must be executed with `wasmtime` and the `--dir` flag to grant permissions to write to the `target` directory.
The following target in the `justfile` will automate this process:

```make
bench:
  #!/usr/bin/env bash
  benchmarks="$(cargo bench --target wasm32-wasi --features=bench --no-run --color=always 2>&1 | tee /dev/tty | grep -oP 'target/.*.wasm')"

  echo "$benchmarks" \
    | xargs -I{} wasmtime --dir $PWD/target::target {} --bench --color=always
```

Since the benchmarks are now able to store the json results in the `target` directory, `criterion` is able to compare the result with previous runs.

![example of benchmark output](/img/learnings-from-developing-a-zellij-plugin/bench.png)

This technique allowed me to improve the performance of zjstatus a lot, after I learned how to better utilize the borrow checker and be aware of memory allocations.
To improve performance, I tried to avoid `.clone()` calls where possible and instead use references, since references are cheap to copy, whereas `.clone()` needs to allocate memory of the size of the cloned object and copy the data.
