+++
title = "Using the Nickel.rs Router Macro"
path = "/2015/01/05/hypermedia-api-using-rustlang-nickel.html"

[taxonomies]
tags=["rustlang"]
+++

The [nickel.rs Web Application Framework for Rust](http://nickel.rs/) is inspired by the popular [node.js express](http://expressjs.com/) framework. The stated goal of the nickel.rs framework is to show people that it can be easy to write web servers using a sytems language like Rust. I have been using the framework to create hypermredia examples using my hal-rs library.


One of the downsides to a systems language like Rust is the verbosity of the syntax. Someone used to writing in Python or Ruby may be in for quite a shock. I started really feeling this when writing route handlers used by nickel. Here is the simple example from the docs:

```rust
fn a_handler (_request: &Request, response: &mut Response) {
    response.send("hello world");
}

server.get("/", a_handler);
```

Notice the `_request` variable has a leading underscore. This tells the compiler not to throw a warning if this variable is unused. If you do decided to use `_request` later on, then you need to remember to change the variable name to `request`. You also need to make a name for the function so it can be referenced in the call to `server.get`. This sort of boilerplate stuff is not what I want to spend time worrying about. What I really want is a nice looking DSL to describe my routes.

After some perusing of the provided [examples](https://github.com/nickel-org/nickel.rs/tree/master/examples) in nickel.rs, I discovered the `router!` macro. We can use the `router!` macro to get a DSL-like syntax for routing. Here is the same example above using the `router!` macro:

```rust
router! {
    get "/" => |request, response| {
        response.send("hello world");
    }
}
```

When we use the `router!` macro, it expands into the same Rust code as in our first example. We don't have to think of a name for the function, worry about type of request or response or or type out the `server.get` line. If you want to see the `router!` macro used in a real world example, check out the [index response](https://github.com/hjr3/hal-rs-demo/blob/4d0a0ab7a1f69708f0c8a5fa2d6669bed223c67f/src/main.rs#L138-168) from my [hal-rs-demo](https://github.com/hjr3/hal-rs-demo/) web server. The code for the `router!` macro is [here](https://github.com/nickel-org/nickel.rs/blob/b8bb31d0efe47f105f6701f73efe0ecd4a6c83de/nickel_macros/src/macro.rs).

The jury is still out on whether or not nickel.rs, or even Rust itself, will be suitable for creating web servers that serve up API responses and HTML to clients. I like many things about Rust though, so I will continue to find out.
