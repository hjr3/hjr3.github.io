---
layout: post
title: "The _with Function Pattern in Rust"
tags:
- rustlang
status: publish
type: post
published: true
---

I really like \_with style functions that accept a `FnOnce` callback. The scoping rules work out really well when using these functions. I was working with the [slab][slab] crate recently and used the [Slab#insert_with][Slab#insert_with] function. This function takes a callback where an object is supposed to be allocated before being inserted into the slab. The function returns an `Option` type. I was trying to figure out to _drop_ the newly created object if the function returned `None` (meaning the insertion failed). After a few minutes it dawned on me that the object was out already dropped!

Example:

```rust
extern crate slab;

#[derive(Debug)]
struct MyType {
    index: usize,
    value: String
}

type Slab = ::slab::Slab<MyType, usize>;

fn main() {

    let mut slab: Slab = Slab::new(128);

    let f = |index: usize| -> MyType {
        MyType {
            index: index,
            value: "a very very very long string".to_string()
        }
    };

    match slab.insert_with(f) {
        Some(index) => {
            println!("Inserted MyType at index {}", index);
        },
        None => {
            // If insertion fails, `MyType` will go out of scope and be dropped/freed.
            println!("Failed to insert into slab");
        }
    }
}
```

The newly allocated `MyType` is _moved_ from the callback into the `Slab#insert_with` scope. If the insert fails, then `Slab#insert_with` returns `None`. The newly allocated type is left within the `Slab::insert_with` function scope. Once `Slab#insert_with` returns, the newly allocated type will be automatically dropped. When an object is dropped, the destructor is called and any allocated memory will be freed.

[edit: An explanation of drop semantics can be found [here][drop semantics].]

The slab crate is an elegant little library that allocates a chunk of memory on the heap and stores values using a custom type for the index. It incorporates a lot of the core Rust concepts. I am learning a lot by studying the code.

[slab]: https://github.com/carllerche/slab
[Slab#insert_with]: https://github.com/carllerche/slab/blob/master/src/lib.rs#L142
[drop semantics]: https://github.com/rust-lang/rfcs/blob/master/text/0320-nonzeroing-dynamic-drop.md#appendices
