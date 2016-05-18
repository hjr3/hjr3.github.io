---
layout: post
title: "Creating a basic webservice in Rust"
tags:
- rustlang
status: publish
type: post
published: true
---

In this post I am going to walk through the creation of a webservice in Rust. This is a _Getting Started_ post that will serve as a foundation for future posts. The webservice will return a static json response to start. There are a few different options for web frameworks in Rust, but practically all of them use the underlying HTTP library called [hyper](https://crates.io/crates/hyper). I am most familiar with [nickel](http://nickel.rs/), so we will be using that. Once the code is complete, we will be creating a release build that is a completely static (standalone) binary. We will then be able to deploy this binary on any modern Linux distro, including Ubuntu and Alpine Linux.

Before we get into any real code, I want to document the environment I am using so you can follow along. I am using a MacBook Air with OS X version 10.11.4. I installed Rust using [rustup.rs](https://www.rustup.rs/) and am using the current stable Rust version 1.8.0. At the time of this writing, rustup is in beta. However, it is quite stable and will soon be the official way to install Rust. I will not go into detail on how to install rustup. Please see the official documentation for that. Finally, I will be using a docker container to build a static binary using musl. I will be doing all development on my laptop and only using the docker container and musl to create a _release_ build.

Mac OS X information:

```
$ sw_vers
ProductName:	Mac OS X
ProductVersion:	10.11.4
BuildVersion:	15E65
```

Rust version (and toolchain):

```
$ rustup show
stable-x86_64-apple-darwin (default)
rustc 1.8.0 (db2939409 2016-04-11)
```

Docker version:

```
$ docker -v
Docker version 1.11.1, build 5604cbe
```

I mentioned earlier that we are going to using a web framework that is based on the hyper crate. The hyper crate supports TLS/SSL using the OpenSSL library. Unfortunately, Mac OS X 10.11 switched to using LibreSSL instead. In order to compile our webservice, we need to first install OpenSSL. While annoying, this is trivially easy using homebrew.

```
$ brew install openssl
$ brew link --force openssl
$ openssl version
OpenSSL 0.9.8zh 14 Jan 2016
```

Note: If you really do not want to install OpenSSL, you can build _debug_ versions on the docker container too. Build times will be slow and testing/debugging will be significantly more difficult.

Now we can finally get to coding. Let us start by creating a new project using Cargo.

```
$ cargo new --bin demo
$ cd demo
$ cargo build
Compiling demo v0.1.0 (file:///Users/herman/projects/demo)
```

Now we need to add the [nickel](https://crates.io/crates/nickel) web framework so we can accept HTTP requests and deliver responses. We can be a little less conservative than crates.io when specifying my dependencies. None of these crates are 1.0 yet, but they are being developed and maintained. We do not want to wildcard (`"*"`) the entire version number for each dependency as that will leave our webservice suscepitble to backwards compatibility breaks in the future. Leaving the _patch version_ a wildcard will allow us to update the libraries when they have bug fixes in the future without risking a major backwards compatibility break. **Edit: As was explained to me, cargo will update to the latest compatible version (in accordance with SemVer) by default. So `0.8.1`, `0.8.*` and `^0.8.1` all mean the same thing. The standard convention is to specify the version shown to you on crates.io. I have updated the below `Cargo.toml` example.**

```
$ cat Cargo.toml
[package]
name = "demo"
version = "0.1.0"
authors = ["Your Name <yourname@example.com>"]

[dependencies]
nickel = "0.8.1"
$ cargo build
   Updating registry `https://github.com/rust-lang/crates.io-index`
   Compiling regex-syntax v0.3.1
   ...
   Compiling demo v0.1.0 (file:///Users/herman/projects/demo)
```

When we `cargo build`, we will download and compile 45 total crates into our webservice. This will also create a [Cargo.lock](https://doc.rust-lang.org/book/getting-started.html#what-is-that-cargolock) file that contains exact information about our dependencies. Since we are building a binary/executable, we will commit the lock file with the rest of our code.

Now we can open up `src/main.rs` and start creating our service. In this first step, I will show how to write a handler for an HTTP GET request that generates a very simple json response.

```rust
#[macro_use] extern crate nickel;

use nickel::{Nickel, MediaType};

fn main() {

    let mut server = Nickel::new();

    server.utilize(router! {
        get "/foo" => |_request, mut response| {
            response.set(MediaType::Json);
            r#"{ "foo": "bar" }"#
        }
    });

    server.listen("127.0.0.1:6767");
}
```

Let us walk through the above code. We start by importing the nickel crate. The nickel crate includes macros and in order to use them we need to put `#[macro_use]` in front of the import statement. We then need to specify which parts of the nickel crate we want to use. It is valid to put `use nickel::*`, but I prefer to be explicit about which parts of a crate I am using.

Our main function is the entry point of our program. We create a new server object, declare what routes we want to handle and then start listening for requests. The design of nickel is similar to the [Express](http://expressjs.com/) node.js framework. The `server.utilize` function is used to register middleware with the server object. Using the `router!` macro, we can specify each route we want to handle. To start, we want to accept an HTTP GET request for `/foo`. We can now specify how we want to handle that request inside of a lambda function.

The lambda function includes `request` and `response` parameters. Since we are not looking at any information from the request, we prepend it with an underbar (`_`) so the compiler does not throw a warning. We will be modifying the response to set the media type, so we prepend that with `mut`. We then set the media type for the response as json and create a literal json string to return. The `r#...#` syntax is the [raw literal string notation](https://doc.rust-lang.org/reference.html#raw-string-literals) in Rust.

We now have a working webservice. Let us see it in action.

Start the server:

```
$cargo run
   Compiling demo v0.1.0 (file:///Users/herman/projects/demo)
   Running `target/debug/demo`
Listening on http://127.0.0.1:6767
Ctrl-C to shutdown server
```

Make a request in another terminal:

```
$ curl --silent localhost:6767/foo | python -mjson.tool
{
    "foo": "bar"
}
```

The last thing we will do is create a static binary that can run on a modern Linux server. This is known as [cross-compiling](http://blog.rust-lang.org/2016/05/13/rustup.html) and it is becoming a first-class feature of the Rust ecosystem. By default, compiling a Rust program to run on Linux has a few dynamic dependencies. There are many pros and cons to the _static vs dynamic_ debate, but in this example I want to make the webservice completely static so I can deploy it without relying on the presence of any dynamic libraries. Mac OS X uses clang instead of gcc. In order to use musl, we will need gcc. I am going to use a docker container rather than install gcc. At the time of this writing, I am using Docker for Mac (beta). It should not matter how you have docker running on OS X though. I am going to use the [rust-musl-builder](https://github.com/emk/rust-musl-builder) docker container, which was built specifically for this purpose.

If you do not have the container installed, running the below command will first pull that container. Annoyingly, it will not execute the command after it downloads the container.

```
$ docker run --rm -it -v "$(pwd)":/home/rust/src ekidd/rust-musl-builder cargo build --release
Unable to find image 'ekidd/rust-musl-builder:latest' locally
latest: Pulling from ekidd/rust-musl-builder
...
Digest: sha256:0c2e9357d1cff9fc9c37396953749ca601fe4d3ee1b47104cd46d99a1a90f576
Status: Downloaded newer image for ekidd/rust-musl-builder:latest
$ docker run --rm -it -v "$(pwd)":/home/rust/src ekidd/rust-musl-builder cargo build --release
    Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading nickel v0.8.1
 ...
    Compiling utf8-ranges v0.1.3
    ...
    Compiling demo v0.1.0 (file:///home/rust/src)
$ ls -lah target/x86_64-unknown-linux-musl/release/demo
-rwxr-xr-x  1 herman  staff   2.5M May 16 08:34 target/x86_64-unknown-linux-musl/release/demo
```

Now that we have the container, we can create the release build. It is going to take a bit of time. Rust has to download all the dependencies on that container, compile each of them and then compile our main program. We also specified the `--release` flag so Rust is optimizing each step of the build process.

We now have a 2.5MB statically compiled executable. We can run our webservice on any modern Linux distro just by copying the file there and running it. There is still a lot more to do to make a _production ready_ webservice, but this is the basic foundation that we will refer back to when making future improvements. You can find the complete working example on github at [https://github.com/hjr3/webservice-demo-rs](https://github.com/hjr3/webservice-demo-rs/tree/blog-post-1).
