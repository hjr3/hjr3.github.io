+++
title = "WASI example using Rust and Lucet"
path = "2019/04/01/wasi-example-using-rust-and-lucet.html"

[taxonomies]
tags=["rustlang"]
+++

Lucet is Fastly's [native WebAssembly compiler and runtime][lucet]. Using the Lucet runtime and Rust's `wasm32-unknown-wasi` target, we can create a WASM program that runs on the server.

<!-- more -->

At the time this blog post was written, the `wasm32-unknown-wasi` target is only available on Rust nightly. Make sure you are using a version of nightly that is as recent as April 1, 2019.

```bash
$ rustup update
info: syncing channel updates for 'stable-x86_64-apple-darwin'
info: syncing channel updates for 'nightly-x86_64-apple-darwin'
352.7 KiB / 352.7 KiB (100 %)  80.0 KiB/s ETA:   0 s
info: latest update on 2019-04-01, rust version 1.35.0-nightly (e3428db7c 2019-03-31)
```

Add the `wasm32-unknown-wasi` target using `rustup`:

```bash
$ rustup target add wasm32-unknown-wasi --toolchain nightly
info: downloading component 'rust-std' for 'wasm32-unknown-wasi'
 10.4 MiB /  10.4 MiB (100 %)   1.1 MiB/s ETA:   0 s
info: installing component 'rust-std' for 'wasm32-unknown-wasi'
```

Create a new binary, via Cargo and compile it to `wasm32-unknown-wasi`:

```bash
$ cargo init hello
     Created binary (application) package
$ cd hello/
$ cargo +nightly build --target wasm32-unknown-wasi
   Compiling hello v0.1.0 (/Users/herman/Code/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.59s
```

We now have a `hello.wasm` file that supports [WASI][WASI]. The `hello.wasm` file will not run on its own though. We will use Fastly's Lucet runtime to get our program running. I created a Docker container with Lucet already built at https://hub.docker.com/r/hjr3/lucet. I wrote a [blog post][blog post] on this if you want more details. Use the `hjr3/lucet` container to build native `x86_64` code from our WASM file and then run it using the Lucet runtime:

```bash
$ docker run --rm -it -v "$(pwd)":/usr/local/src hjr3/lucet lucetc-wasi -o hello.so target/wasm32-unknown-wasi/debug/hello.wasm
$ docker run --rm -it -v "$(pwd)":/usr/local/src hjr3/lucet lucet-wasi hello.so
Hello, world!
```

One neat thing about this example is our local development operating system does not have to match our target runtime operating system. We can compile our Rust program locally on MacOS and only use the `hjr3/lucet` docker container (which runs Ubuntu Xenial) to convert/run the program.

WASI is brand new and a lot of development is still going on. From the [Rust PR] that added support for the `wasm32-unknown-wasi` target:

> The wasi target in libstd is still somewhat bare bones. This PR does not
fill out the filesystem, networking, threads, etc. Instead it only
provides the most basic of integration with the wasi syscalls...

I plan on demonstrating more examples as libstd gets built out.

[lucet]: https://www.fastly.com/blog/announcing-lucet-fastly-native-webassembly-compiler-runtime
[WASI]: https://hacks.mozilla.org/2019/03/standardizing-wasi-a-webassembly-system-interface/
[blog post]: /2019/03/31/lucet-in-five-minutes.html
[Rust PR]: https://github.com/rust-lang/rust/pull/59464
