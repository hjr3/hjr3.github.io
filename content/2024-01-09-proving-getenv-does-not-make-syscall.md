+++
title = "Proving getenv Does Not Make a Syscall"

[taxonomies]
tags=["rust", "dtruss"]
+++

I saw this statement in an accepted answer on Stack Overflow:
> Retrieving the value of an environment variable will incur a system call.

[source](https://stackoverflow.com/a/7460584/775246)

This answer surprised me as I did not think this was the case. There is an edit farther down that has links to other Stack Overflow posts saying get `getenv` does not make a syscall. Let us prove it ourselves.

<!-- more -->

Here is a small program that reads an environment variable and prints out the value.

```rust
fn main() {
    let v = ::std::env::var("USER").unwrap();
    println!("USER: {}", v);
}
```

```
$ cargo build
$ ./target/debug/env-syscall-test
USER: herman
```

Now we can use `dtruss` to print out all the syscalls our program makes.

```
$ sudo dtruss ./target/debug/env-syscall-test
dtrace: system integrity protection is on, some features will not be available

SYSCALL(args) 		 = return
USER: root
access("/AppleInternal/XBS/.isChrooted\0", 0x0, 0x0)		 = -1 2
...elided
```

The `dtruss` output shows that our program prints `USER: root` without making any syscall.
