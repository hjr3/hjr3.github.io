---
layout: post
title: "Exploring the Rust Standard Library"
tags:
- rustlang
status: publish
type: post
published: true
---

I was writing some Rust with a colleague when they asked me about the cases where Rust deferences types for us automatically. I said that Rust will [automatically dereference pointers when making method calls][method deref], but otherwise there was no compiler magic. This conflicted with their experience with Rust and presented an example like this:

```rust
let a = 1;
let b = 2;
let c = a + &b;
```

The basic question is: _"How does this code work without us dereferencing `b`?"_. I think this is a great question and touches on an aspect of Rust that I really like.

Rust is written in, well, Rust. While I myself do not understand much of the type checking code, I can read and understand the vast majority of the core and standard libraries. Many languages, especially scripted ones, are not like this. Often times the core and standard libraries are written in C. In order to see how this code works you have to not only understand C, but also understand how the data structures in C relate to your language. The barrier to entry is quite high. The net result is that most people rely on documentation, blogs or stack overflow to understand how major parts of the language work. They cannot go see for themselves. I really like that major parts of _Rust proper_ are much more accessible.

Let us go see for ourselves why we can add a type `T` and a reference to a type `&U`. Along the way we will learn how to explore some of the inner workings of Rust. To start, we need to bring up the [std library documentation][std library doc] webpage. We can then search for the word "[add][add search]" and the first result will be [std::ops::Add][std::ops::Add] with a summary descriptipn of _The Add trait is used to specify the functionality of +_. That seems like a good place to start. We now know that adding two things together is implemented using the `Add` trait. The webpage for the `Add` trait even shows us a simple implementation.

Scrolling down the webpage will list all the implementations of the `Add` trait that exist in the standard library. Looking at 14th item in that list, you will see `impl<'a> Add<&'a usize> for usize`. The standard library has a specific implementation of the `Add` trait for the case when the right hand side (rhs) of the addition is a reference to a type. If you scroll down more you will see that all the numeric types are listed. Each numeric type has implementations of the `Add` trait for `T + U`, `&T + U`, `T + &U` and `&T + &U`. You can also find similar results for subtraction, multiplication and division.

You will find this pattern repeated over and over in Rust. Some generic functionality is represented as a trait. In order to specify that functionality, that trait must be implemented. It is not uncommon to see long lists of implementations for traits in the standard library. While this may appear to be a lot of boilerplate, the benefit is that Rust can check our code at compile time (the alternative would be to wait until runtime which is less safe and makes our code slower).

If we jump back to the list of implementations for the `Add` trait you might notice something interesting. The standard library does not specify implementations for addition between different types. This code will not work:

```rust
let a: u64 = 1;
let b: u32 = 2;
let c = a + b;
```

If you ever ran into a compiler error about adding two different numeric types together before and wondered why that does not work, now you know. It is not some compiler magic, but instead the simple fact that the standard library does not list `impl Add<u32> for u64`.

I mentioned above that we should not have to rely on the documentation to understand how standard features work. So far, we have been relying on the _excellent_ documentation of the Rust standard library. If we scroll back up the webpage, we should see the [\[src\]][Add trait impl] link to the actual source code for the `Add` trait. If we follow that link, we will see the definition of the `Add` trait and then a macro called `add_impl!` being defined. Macros can be a little hard to understand, but if we can generally understand that this macro defines `T + T`. Right below that macro we should see something like:

```rust
add_impl! { usize u8 u16 u32 u64 isize i8 i16 i32 i64 f32 f64 }
```

Now we see how all the numeric types implement the `Add` trait for `T + T`. We need to look a little deeper to understand how references are handled. If we look back at the bottom of the `add_impl!` macro we will see another macro called `forward_ref_binop!`. If we scroll up the page we can find the definition for `forward_ref_binop!` and we will notice that it defines the behavior for `&T + U`, `T + &U` and `&T + &U`. Take note that the use of macros greatly decreased the amount of boilerplate in the Rust standard library. Macros are harder to read, but they are certainly powerful.

I find myself following the above approach when I run into something about Rust I do not understand. This even works for crates listed on crates.io that generate documentation. For example, the [mio crate][mio crate] hosts [documentation][mio doc] on Amazon S3 but the look, feel and functionality are the same as the official Rust documentation webpages. There are other ancillary benefits to exploring the Rust standard library. Along the way you learn things you were not explicitly looking for. The standard library can also be a great reference for how to implement something. The code is written to a high standard and puts a lot of emphasis on correctness. Reading the core and standard libraries may seem daunting at first, especially if you are not familar with macros, but stick with it. With some practice and patience it will become much more familiar to you. At that point, you can start contributing too!

[method deref]: http://stackoverflow.com/a/28552082
[std library doc]: https://doc.rust-lang.org/std/
[add search]: https://doc.rust-lang.org/std/?search=add
[std::ops::Add]: https://doc.rust-lang.org/std/ops/trait.Add.html
[Add trait impl]: https://doc.rust-lang.org/src/core/ops.rs.html#182-190
[mio crate]: https://crates.io/crates/mio/
[mio doc]: http://rustdoc.s3-website-us-east-1.amazonaws.com/mio/v0.5.x/mio/
