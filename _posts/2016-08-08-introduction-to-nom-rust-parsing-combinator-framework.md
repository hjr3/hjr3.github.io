---
layout: post
title: "Introduction to nom: a parsing framework written in Rust"
tags:
- rustlang
status: publish
type: post
published: true
---

This is an introduction to a parsing library called [nom](https://github.com/Geal/nom). The `nom` crate is written by Geoffrey Couprie, aka [Geal](https://github.com/Geal), and is a remarkably complete and powerful library for building parsers. I recently did a lot of parsing of bytes on the wire for my [carp](https://github.com/hjr3/carp-rs) library and it was a lot of work. I wish I had come across the `nom` library before I had done all of that.

The description of `nom` is a _Rust parser combinator framework_ which can sound a little initimdating. Another way of saying this is that `nom` uses a lot of small functions and macros that make parsing code easy to write and read. I will say that `nom` can be a bit intimidating to start using. The API has a lot of surface area to learn and the error messages can be hard to understand. The cryptic error messages are due to the use of macros and not anything specific to nom. While it can take a little bit of effort to get started using `nom`, but I think it is well worth it.

## Parsing Text

The `nom` library can parse pretty much anything, but let us start with text. When parsing text, we might be tempted to reach for something like regular expressions. This is an alternative approach that leverages Rust's typing. Also, `nom` is probably faster and more efficient than any regular expression we might write. The first thing to understand about `nom` is that it only deals in byte arrays (`&[u8`). Our text to parse will most likely be in the form of a string. We can convert a string to a byte array using `.to_bytes().` To get usuable results from our parser, we must convert (or map) a matched sequence of bytes into the type that we want. Knowing this, let us start looking at how to parse text input.

The bread and butter of our parsing is going to be the use of the `tag!` and `map_res!` macros. The `tag!` macro consumes the specified string from the byte array. For example, if we had a string of `"hello Herman"`, we would specify `tag!("hello")` to parse out the first word. The `tag!` macro works great when we know what string we want to match. It does not work for dynamic strings. The `map_res!` macro will be used for dynamic input.

We have to write a more abstract parser for dynamic strings. We can parse the `"Herman"` part of the string using the `alpha` function. The `alpha` function is provided by `nom` and will return the longest list of alphabetic characters it finds as a byte array. If it is dynamic, it probably means this is part of the input we want to capture. Getting back a byte array of `&['H', 'e', 'r', 'm', 'a', 'n']` is not ideal work with. We want to convert that into a string. Using `map_res!` we can map (convert) the byte array into a string: `map_res!(alpha, std::str::from_utf8)`. The `map_res!` (map result) macro is known as a combinator. We are _combining_ the `alpha` function with the [std::str::from_utf8](https://doc.rust-lang.org/std/str/fn.from_utf8.html) function. The `std::str::from_utf8` function is part of the Rust standard library and converts a slice of bytes (or a byte array) into a UTF8 encoded string. So `map_res!(alpha, std::str::from_utf8)` is saying that we want to grab the longest array of alphabetic characters and then we want to pass that byte array of alphabetic characters to the `std::str::from_utf8` function.

Now that we can parse both parts of `hello Herman`, we can put it all together into a more complex parser:

```rust
#[macro_use]
extern crate nom;

use nom::{IResult, space, alpha, alphanumeric, digit};

named!(name_parser<&str>,
    chain!(
        tag!("hello") ~
        space? ~
        name: map_res!(
            alpha,
            std::str::from_utf8
        ) ,

        || name
    )
);
```

In the above example, we are using the `named!` macro to create a parser function named `name_parser`. We specify the `&str` type as the return type of our parser. We use the `chain!` combinator macro to apply a series to parsers and assemble their results. We use the `~` character as the separator between parser functions/macros and a `,` to denote the end of the parser chain. The last part of the `chain!` combinator takes a closure, `|| name`, where we can use the previously defined `name` variable. We now have a function `name_parser` that accepts a string that begins with `hello`, has one or more spaces and then contains a series of alpha characters. We map those alpha characters into a string and assign that value to a variable called `name`. Finally, we return `name` from the closure, which will be the `&str` our function returns. Here is a test case proving it:

```rust
#[test]
fn test_name_parer() {
    let empty = &b""[..];
    assert_eq!(name_parser("hello Herman".as_bytes()), IResult::Done(empty, ("Herman")));
    assert_eq!(name_parser("hello Kimberly".as_bytes()), IResult::Done(empty, ("Kimberly")));
}
```

Notice that the `name_parser` function does not actually return a `&str`. It actually returns an `IResult` type that represents whether the parsing is `Done`, `Incomplete` or an `Error`. If the parsing was successful, the result will be `IResult::Done(input_remaining, output)`. In our above test, there is no more input left so the byte array is empty. The _output_ is our `&str` containing the dynamic name.

This might seem like a lot of work for such a basic parser. However, this is building a foundation for creating a lot more complex parsers. For example, we can now create a parser to convert a string of numeric characters into a number:

```rust
// Parse a numerical array into a string and then from a string into a number
named!(usize_digit<usize>,
    map_res!(
        map_res!(
            digit,
            std::str::from_utf8
        ),
        std::str::FromStr::from_str
    )
);
```

And we can even go a step further and separate the parsing of a numerical array into a smaller parser called `numeric_string`. We can then map the result of `numeric_string` into a `usize` type in the `usize_digit` parser function:

```rust
named!(numeric_string<&str>,
    map_res!(
        digit,
        std::str::from_utf8
    )
);

named!(usize_digit<usize>,
    map_res!(
        numeric_string,
        std::str::FromStr::from_str
    )
);
```

Now that we have a generic parser to parse numerical arrays, we can also create a parser to parse a string into a `u64` using the same `numeric_string` function defined above:

```rust
named!(u64_digit<u64>,
    map_res!(
        numeric_string,
        std::str::FromStr::from_str
    )
);
```

You start to see how powerful the combination of small parsers can be. Not only does `nom` make it easy to write parsers, I think it also makes it easy to read parsers later and understand what they are doing. It also makes it easier to write tests against smaller parsers to verify their correctness. We have just scratched the surface of what `nom` can do. There are many other [parsers](http://rust.unhandledexpression.com/nom/#functions) and [combinators](http://rust.unhandledexpression.com/nom/#macros) available. There are also a number [example](https://github.com/Geal/nom/issues/14) [parsers](https://github.com/Geal/nom/tree/master/tests) to use as a reference.
