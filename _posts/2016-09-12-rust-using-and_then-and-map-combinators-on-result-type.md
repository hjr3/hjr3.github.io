---
layout: post
title: "Using and_then and map combinators on the Rust Result Type"
tags:
- rustlang
status: publish
type: post
published: true
---

If you have spent any amount of time learning Rust, you quickly become accustomed to `Option` and `Result` types. It is through these two core types that we make our programs reliable. My background is with C and dynamic languages. I found it easiest to use the `match` keyword when working with these types. There are also combinator functions like `map` and `and_then` which allow a set of computations to be chained together. I like to chain combinators together so error logic is separated from the main logic of the code.

I recently returned home from RustConf 2016 where the [futures](https://github.com/alexcrichton/futures-rs) crate had a [0.1.1](https://crates.io/crates/futures/0.1.1) release along with the first glimpses of [tokio](https://github.com/tokio-rs/tokio). All futures implement a `poll` function that returns a [Poll](https://github.com/alexcrichton/futures-rs/blob/9bd186bef3430d26747ee886c54d5e68e0405275/src/lib.rs#L354) type. The `Poll` type is defined as `pub type Poll<T, E> = Result<Async<T>, E>;`. Thus, if we want to use futures, we need to be comfortable with combinator functions implemented on the core `Result` type. You will not be able to fall back on using the `match` keyword.

### Approach

I will be providing explicit types throughout the examples to make it easier to understand what is happening. In the vast majority of cases, you can let compiler infer the types. In fact, it is idiomatic to let the compiler infer the types.

The [Standard Library API Reference](https://doc.rust-lang.org/std/result/enum.Result.html#method.and_then) for `Result` combinators does a good job with explanations and simple examples. However, most examples use the same type for both the `Ok` and `Err` variants. I think this makes it harder to understand what is going on. I will be using a `Err(&'static str)` variant the examples so I can use easy to identify error messages. If the `'static` lifetime confuses you, know that `&'static str` means hard-coded string literal. Example: `let foo: &'static str = "Hello World!";`.

## `and_then` Combinator

Let us start with the `and_then` combinator function. The `and_then` combinator is a function that calls a closure if, and only if, the variant of the `Result` enum type is `Ok(T)`.


```rust
let res: Result<usize, &'static str> = Ok(5);
let value = res.and_then(|n: usize| Ok(n * 2));
assert_eq!(Ok(10), value);
```

In this first example, the value of `res` is `Ok(5)`. Per our definition of `and_then`: `and_then` will match on the `Ok` variant and call the closure with the `usize` value of 5 as the argument. What happens if `res` is an `Err` variant?

```
let res: Result<usize, &'static str> = Err("error");
let value = res.and_then(|n: usize| Ok(n * 2)); // <--- closure is not called
assert_eq!(Err("error"), value);```
```

In this second example, the value of `res` is `Err("error")`. Per our definition of `and_then`: `and_then` will match on the `Err` variant and _skip_ calling the closure. The value of `Err("error")` will be returned as is. This is convenient as we were able to write a closure that ignored errors. The value `Err("error")` will be passed along in the background to the end of the combinator chain. So far we have only been returning `Ok` from closure. Our closure can also return an `Err` too.

### Chaining Multiple `and_then` Functions

Instead of multiplying, let us divide the result by 2. To protect against division by zero errors, we need to add another step in the chain that will return an error if the value is zero.

```rust
let res: Result<usize, &'static str> = Ok(0);

let value = res
    .and_then(|n: usize| {
        if n == 0 {
            Err("cannot divide by zero")
        } else {
            Ok(n)
        }
    })
    .and_then(|n: usize| Ok(2 / n)); // <--- closure is not called

assert_eq!(Err("cannot divide by zero"), value);
```

The initial value of `Ok(0)` will be passed to the first closure. In this case, `n` does equal `0` and the closure returns `Err("cannot divide by zero")`. Our next call to `and_then` identifies that we now have an `Err` variant of `Result` and does not call the closure.

### Flattening Results

There are times when we have nested `Result` types. It is generally a good strategy to try and flatten the result out. For example, we can flatten `Result<Result<usize, &'static str>, &'static str>` to `Result<usize, &'static str>`. A flatter `Result` is generally easier for later code to deal with.

```rust
let res: Result<Result<usize, &'static str>, &'static str> = Ok(Ok(5));

let value = res
	.and_then(|n: Result<usize, &'static str>| {
		n // <--- this is either Ok(usize) or Err(&'static str)
	})
	.and_then(|n: usize| {
		Ok(n * 2)
	});

assert_eq!(Ok(10), value);
```

In the above example, the first `and_then` closure is returning `n`. Note that in previous examples, we were wrapping our return value in either the `Ok` or `Err` variant of the `Result` enum. In this example, our goal is to flatten the result so we will not explicitly return `Ok` or `Err`. The value of `n` is going to be either `Ok(usize)` or `Err(&'static str)`. As such, we can return `n` as it is. If the value of `n` is of type `Ok(usize)` then the value will be passed to the next `and_then` as expected. If the value of `n` is of type `Err(&'static str)` then the second `and_then` function will be bypassed.

The `and_then` function is called flatMap in scala and you can see why. We are flattening the type from `Result<Result<_, _>, _>` to `Result<_, _>` by _mapping_ variants in the internal `Result` to the outer `Result`.

## `map` Combinator

So far we have been using `and_then` to combine computation and flatten our nested `Result`s. The examples have been using `Result`s with types that are the same types we wanted to end up with. Sometimes we are given a `Result` where one or both variants are not the type we want. We will use `map` to transform one `Result` type into another.

### Basics

If you primarily use a dynamically typed language, you may have used `map` as a replacement for iterating/looping over a list of values. We can do this same thing in Rust too.

```rust
let res: Vec<usize> = vec![5];
let value: Vec<usize> = res.iter().map(|n| n * 2).collect();
assert_eq!(vec![10], value);
```

Using `map` with a `Result` type is a little different. The `map` function calls a closure if, and only if, the variant of the `Result` enum is `Ok(T)`. Here is our very first `and_then` example, but using `map` instead.

```rust
let res: Result<usize, &'static str> = Ok(5);
let value: Result<usize, &'static str> = res.map(|n| n * 2);
assert_eq!(Ok(10), value);
```

This looks very similar to the first `and_then` example, but notice that we returned `Ok(n * 2)` in `and_then` example and we are returning `n * 2` in this example. The `map` function _always_ wraps the return value of the closure in the `Ok` variant.

### Mapping the `Ok` Result Variant To Another Type

Let us look at an example where the `Ok(T)` variant of the `Result` enum is of the wrong type. Example: We are given `Result<i32, _>`, but we want `Result<usize, _>`.

```rust
let given: Result<i32, &'static str> = Ok(5i32);
let desired: Result<usize, &'static str> = given.map(|n: i32| n as usize);

assert_eq!(Ok(5usize), desired);

let value = desired.and_then(|n: usize| Ok(n * 2));

assert_eq!(Ok(10), value);
```

In this example, the value of `res` is `Ok(5i32)`. Per our definition of `map`, `map` will match on the `Ok` variant and call the closure with the `i32` value of 5 as the argument. When the closure returns a value, `map` will wrap that value in `Ok` and return it.

If the given value is an `Err` variant, it is passed through both the `map` and `and_then` functions without the closure being called.

```rust
let given: Result<i32, &'static str> = Err("an error");
let desired: Result<usize, &'static str> = given.map(|n: i32| n as usize); // <--- closure not called

assert_eq!(Err("an error"), desired);

let value = desired.and_then(|n: usize| Ok(n * 2)); // <--- closure not called

assert_eq!(Err("an error"), value);
```

### Mapping Both Variants of Result

What if both variants of the Result were different? Example: We are given `Result<i32, MyError>`, but we want `Result<usize, &'static str>`.

We only transform the `Ok(i32)` variant in the above example. In this example, we will need to also transform the `Err(MyError)` variant into `Err(&'static str)`. In order to do this, we will need to use `map_err` to handle the `Err(E)` variant. The `map_err` combinator function is the opposite of `map` because it matches only on `Err(E)` variants of `Result`.

```rust
enum MyError { Bad };

let given: Result<i32, MyError> = Err(MyError::Bad);

let desired: Result<usize, &'static str> = given
    .map(|n: i32| {
       n as usize
    })
    .map_err(|_e: MyError| {
       "bad MyError"
    });

let value = desired.and_then(|n: usize| Ok(n * 2));

assert_eq!(Err("bad MyError"), value);
```

You must understand that:

   * `map` only handles the `Ok(T)` variant of `Result`
   * `map_err` only handle the `Err(E)` variant of `Result`

## Different Return Types Using `and_then` And `map`

The `and_then`, `map` and `map_err` functions are not constrained to return the same type inside their variants. The `map` functions can be given `Ok(T)` and return `Ok(U)`. The `map_err` function can be given `Err(E)` and return `Err(F)`. The `and_then` function can be given `Ok(T)` and return `Ok(T)` or `Err(F)`!

Let us try a complicated example where we are given a nested Result, but none of the types match the desired types we want. Example: We are given `Result<Result<i32, FooError>, BarError>`, but we want `Result<usize, &'static str>`.


```rust
enum FooError {
    Bad,
}

enum BarError {
    Horrible,
}

let res: Result<Result<i32, FooError>, BarError> = Ok(Err(FooError::Bad));

let value = res

    // `map` will only call the closure for `Ok(Result<i32, FooError>)`
    .map(|res: Result<i32, FooError>| {

        // transform `Ok(Result<i32, FooError>)` into `Ok(Result<usize, &'static str>)`
        res
            // transform i32 to usize
            .map(|n: i32| n as usize)

            // transform `FooError` into `'static str`
            .map_err(|_e: FooError| "bad FooError")

    })

    // `map_err` will only call the closure for `Err(BarError)`
    .map_err(|_e: BarError| {
        // transform `BarError` into `'static str`
        "horrible BarError"
    })

    // `and_then` will only call the closure for `Ok(Result<usize, &'static str>)`
    // Note: this is result of our first `map` above
    .and_then(|n: Result<usize, &'static str>| {
        // transform (flatten) `Ok(Result<usize, &'static str>)` into `Result<usize, &'static str>`
        // this may be `Ok(Ok(usize))` _or_ `Ok(Err(&'static str))`
        n
    })

    // `and_then` will only call the closure for `Ok(usize)`
    .and_then(|n: usize| {
        // transform Ok(usize) into Ok(usize * 2)
        Ok(n * 2)
    });

assert_eq!(Err("bad FooError"), value);
}
```

I decided to inline the explanation into the comments in an effort to make things as clear as possible. You can see how quickly things get complicated. It is my general strategy to try and flatten the nested `Result` out as early as possible to simplify later combinators.

## Conclusion

A lot of functions return `Result` to represent the happy-path value and the error case. Using combinators can help isolate error handling from normal computation. Combinators also allow us to pass along errors all the way to the end. I like the [Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/) for a good visualization of this concept. All the examples we went through work on the `Option` type too. You should now be better equipped to read other code that uses `Result` combinator functions and writing them yourself.


## Extras

### `or_else` Combinator

The `or_else` function combinator is the opposite of `and_then`. It only calls the closure if the result is `Err(E)`. I do not find myself using `or_else` as often as `and_then`. Please feel free to show me what I am missing.

### Debugging Complex Combinators

I like to make types explicit when trying to get a complex combination working. However, this can get unrealistic when dealing with iterators or futures that become deeply nested. When that happens, I start assigning results to incorrect types. Here is a small example, assuming I am confused as to what type `res` is:

```rust
// assume it is not clear what type `res` is
let res: Result<usize, &'static str> = Ok(5);
let c: u8 = res;
```

Which generates:

```
error[E0308]: mismatched types
 --> <anon>:5:13
  |
5 | let c: u8 = res;
  |             ^^^ expected u8, found enum `std::result::Result`
  |
  = note: expected type `u8`
  = note:    found type `std::result::Result<usize, &'static str>`
```

I normally use the variable `c` because I want to _see_ the type of `res` in the compiler error message. Haha, I know.

Here is an example using it in a combinator:

```rust
let res: Result<usize, &'static str> = Ok(5);
let value = res.and_then(|wut| {
    let c: u8 = wut;
});
```

Which generates:

```
error[E0308]: mismatched types
 --> <anon>:6:17
  |
6 |     let c: u8 = wut;
  |                 ^^^ expected u8, found usize

error[E0308]: mismatched types
 --> <anon>:5:32
  |
5 | let value = res.and_then(|wut| {
  |                                ^ expected enum `std::result::Result`, found ()
  |
  = note: expected type `std::result::Result<_, &str>`
  = note:    found type `()`

error: aborting due to 2 previous errors
```

The compiler errors show both the expected input and expected output. I find this really useful when I get lost in all the combinators.

#### Nightly Error Format

As of this writing, Rust `1.11.0` is the stable version. Rust `1.11.0` does not have the new error format that is present in Rust nightly. If I am struggling on a compiler error, I often switch over to using Rust nightly until I solve the error. [Rustup](https://rustup.rs/) makes this easy.

In your current working directory:

   * Switch to nightly - `rustup override set nightly`
   * Switch to stable - `rustup override set stable`
