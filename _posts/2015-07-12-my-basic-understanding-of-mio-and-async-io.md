---
layout: post
title: "My Basic Understanding of mio and Asynchronous IO"
tags:
- rustlang
status: publish
type: post
published: true
---

I needed async IO for a Rust project I was working on. My server needs to read some bytes from a client and then send those bytes back to all registered clients. I decided to use [mio][mio], the Rust async IO library, to build a server. All the examples I found showcased reading from a socket and then writing back to that same socket. Also, many of the examples had caveats about unhandled edge cases and used a lot of `unwrap`. Over the next three posts, I am going to walk through everything I learned about async (or evented) IO as it relates to mio, how I setup my server and talk in depth about some of the ways I thought about handling errors. Let us start with the an overview of how async IO works within the context of mio.

## How mio Exposes Async IO

I initially thought the mio library was a Rust wrapper around [libev][libev]. To my surprise, I realized mio is a replacement for libev. The name mio (metal IO) starts to make a lot more sense. The mio library is interfacing directly with [epoll][epoll] if you are on linux and [kqueue][kqueue] if you are on FreeBSD (or OS X). The mio event loop interface expects us to know how the epoll/kqueue implementations work. Mio gives us complete control over how the async IO works, but it also means there is quite a bit to learn. After a decent amount of a reading, I believe I have a basic understanding of what some of the affordances mio exposes actually do.

### Registration

The `EventLoop` object provided by mio is our main point of contact. Interaction with the event loop is in the form of the `register`, `register_opt`, `reregister` and `deregister` functions. These functions allow our code to control how the event loop interacts with the incoming client connections. All the functions, with the exception of deregister, require four arguments: a `TcpStream` socket for our client connection, a `Token` to identify connections, an `EventSet` to control what events we are notified of and a `PollOpt` to determine how we should be notified. The deregister function only needs the socket.

Understanding how these functions work are key to understanding how to use mio. The concept of registering with an event loop is fairly straight forward, especially once you start looking at the code. It is the arguments to these registration functions that are more dense. I am going to assume you understand the basics of how a socket works and how to read and write bytes using that socket. The other arguments require some more in-depth discussion.

### Tokens

I found the use of tokens strange at first. I am more familiar with the use of callbacks to deal with asynchronous events. Mio uses tokens as an alternative to callbacks in order to achieve the design goal of zero allocations at runtime. The `Token` type is really just a [tuple struct][tuple struct] wrapper around `usize`. This means it is cheap to compare and copy. This will be important later when we start using the `Token` type.

A `Token` is used to identify the state related to a connected socket. We register with the event loop using a token. Later on, the event loop will specify this token when notifying us of an event. A feedback loop of sorts is created. The `Token` is stored, along with the connection state, in the _connection slab_. I am going to discuss the connection slab when we start looking at the code. Trying to explain it without code feels overly complicated.

### EventSet (Formerly Known As Interest)

The `EventSet` object represents the set of events we are interested in being notified of. Until recently, the `EventSet` type was name `Interest`. The `0.3.x` branch still refers to it as `Interest`. There are four types of events:

   * readable - Tells the event loop we want to read data from a client connection.
   * writable - Tells the event loop we want to write data to a client connection.
   * hup - Tells the event loop that we want to be notified when a client closes the connection (hangs up).
   * error - Tells the event loop that we want to listen for errors.

If you are curious as to why `Interest` was renamed to `EventSet`, you can read the [full discussion][discussion] that ultimately resulted in the change. Essentially, epoll and kqueue have slightly different interfaces and this change made it easier for people using the mio library to handle those differences. The `EventSet` type removed the notion of a _read hint_ that is present in many mio examples currently out there.

#### Write Notifications

One thing that confused me a lot was _when_ to use writable. A lot of the examples are just echoing back what the client sent them or they are performing some very simple task. In these cases, you do not need to register `EventSet::writable()` if you want to immediately write back to the socket you just read from. You can just perform the write as part of the current _readable_ event. That being said, you have to be aware that you are blocking the event loop during this time. If you are performing an expensive task between the read and write, you may want to handle this differently. Whether you are reading or writing from the socket, you also need to be aware that the kernel might block you.

#### I Would Block You

[edit: I updated this section to replace the word _block_ with more correct language like _reject_ and _not ready_.]

Even though we are using asynchronous IO, the kernel is not always ready for our reads and writes. When the kernel's internal send or receive buffers are full and it needs to flush them we will be asked to try again. Typically, the kernel communicates _try again_ to us in the form of an error. In Rust, we have `std::io::ErrorKind::WouldBlock`. In C, it is referred to as `EAGAIN`. This error is the kernel's way of letting us know it is not ready for our read or write and they we need to try again. This `WouldBlock` error _must_ be handled. In mio, we are provided with the traits, `TryRead` and `TryWrite`, which catch `WouldBlock` and treat it as a 0 byte read. These traits are convenient to use as our error handling can now assume any `Err(_)` is an unexpected error. More about this when we get to some code samples.

You might be wondering what it means to _try again_ when the kernel is blocking our read or write. In order to understand this, we need to understand the polling options.

### Poll Options

The `PollOpts` exposed by mio really tripped me up at first because I did not understand how epoll/kqueue worked at all. There are basically two different polling options, or _triggers_, we can use. By default, mio will specify `PollOpt::level()` when registering with the event loop. Level-triggered polling is what you would expect from a straight-forward polling implementation. If you are familiar with `select()` in C, this is basically the same thing. The downside to level-triggered polling is that we are expected to handle the events immediately. If we do not handle them immediately, then the event loop will notify us constantly of the event and we end up wasting resources.

What most people opt for is edge-triggered, `PollOpt::edge()`, polling. Edge-triggred polling means that when we receive a read or write event, the event loop will automatically deregister our connection. This means we can get notified of an event and then have the option of handling it now or later. If more events come in for that connection, the event loop will queue those up for us until we register again. This requires us to have to manage the state of our connections, but gives us the flexibility we really want.

We can also combine edge-triggered polling with another option: `PollOpt::oneshot()`. Not only does this option sound super cool, it also guarantees that only one thread will be woken up. This allows us to be thread-safe when reading or writing. Thread safely unlocks the ability for allows us to write multi-threaded epoll processes on top of mio. For my server, I decided to register connections using `PollOpt::edge() | PollOpt::oneshot()` when registering with the event loop.

#### Trying Again

Now that we are familiar with what events we can be notified of and what our polling options are, we need to revisit the notion of trying our read or write again when the kernel _would block_ us. Using edge-triggering, a read or write event means our connection will be deregistered from the event loop. To try again, we need to first save our work and then reregister our connection with the event loop, using our token, so we can be notified after the kernel is done flushing.

## Next Steps

We now have the necessary context to start using mio. It took days for these concepts to really sink in with me. If you grok this already, you are awesome. If not, give it time! I am going to apply these above concepts to actual code in my next post. If you want to get started before my next post, I would start with [test echo server][test echo server] that is part of the mio test suite. There is also the [getting started][mio getting started] documentation that mio provides, though it is somewhat out of date for the `0.4.x` branch.

There are also a few projects that are abstracting a lot of the details needed to get mio working. These can be great example to learn from. The two I have looked at are:

   * [Reactor][Reactor] - Evented polling + network utilities to make life easier 
   * [mioco][mioco] - Allows handling mio connections inside coroutines 

## Sources

In addition to reading the mio soure code and example code, I did a lot of reading about epoll itself. Here is a list of some sources I used to get more familiar with epoll/kqueue:

   * [Overview of epoll options][epoll options overview] - StackOverflow post on the differences between level-triggered and edge-triggered polling
   * [Purpose of edge-triggered polling][purpose of EPOLLET] - StackOverflow post describing the real advantage of edge-triggered polling
   * [Complete epoll example in C][epoll in C] - A blog post walking through an epoll implementation in C. This is great if you are familar with C and not quite as comfortable with Rust.
   * [Event Queues and Threads][Event Queues and Threads] - Detailed document primarily describing Linux's epoll(7) I/O event notification facility as of the 2.6 kernel series.


[mio]: https://github.com/carllerche/mio
[libev]: http://software.schmorp.de/pkg/libev.html
[epoll]: http://man7.org/linux/man-pages/man7/epoll.7.html
[kqueue]: https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man2/kqueue.2.html
[discussion]: https://github.com/carllerche/mio/issues/184
[tuple struct]: https://doc.rust-lang.org/nightly/book/structs.html#tuple-structs
[mio getting started]: https://github.com/carllerche/mio/blob/docs/doc/getting-started.md
[test echo server]: https://github.com/carllerche/mio/blob/master/test/test_echo_server.rs
[Reactor]: https://github.com/rrichardson/reactor
[mioco]: https://github.com/dpc/mioco
[epoll options overview]: http://stackoverflow.com/a/13568962/775246
[purpose of EPOLLET]: http://stackoverflow.com/a/9162805/775246
[epoll in C]: https://banu.com/blog/2/how-to-use-epoll-a-complete-example-in-c/
[Event Queues and Threads]: https://raw.githubusercontent.com/dankamongmen/libtorque/master/doc/mteventqueues
