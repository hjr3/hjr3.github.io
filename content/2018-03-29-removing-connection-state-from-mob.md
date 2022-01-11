+++
title = "Removing Connection State In mob"
path = "2018/03/29/removing-connection-state-from-mob.html"

[taxonomies]
tags=["rustlang", "mio"]
+++

I started writing mob, an multi-echo server using mio, in 2015. I coded mob into a mostly working state and then left it mostly alone, only updating it to work with the latest stable mio. Recently, I started looking at the code again and had the urge to improve it. In a previous [post][post], I talked about managing the state of connections in mob. In this post, I will walk through what I did to remove the need to track connection state. I wanted to remove the state because the implementation required an `O(n)` operation every _tick_ of the mio event loop. It also added a fair amount of complexity to the code.

Before discussing the solution, I want to review the problem I was trying to solve. With asynchronous IO, the state of the connection and the events may get out of sync. I kept running into problems where I was processing events and would discover the connection was no longer present in the slab. This would cause mob to panic instead of resetting that connection and moving on. Here is one example of how this might happen:

   * **mio**: blocks on poll
   * **client**: client sends some data
   * **mio**: receives read event for connection A
   * **mio**: receives write event for connection A
   * **mio**: unblocks and returns 2 events
   * **mob**: read event is processed, there is an error and connection A is removed from slab
   * **mob**: write event is processed, connection A cannot be found and results in a panic

I needed to not panic when `connection A` was not present in the slab. You can read that previous [post][post] for the details on the scheme I concocted to work around this issue. Looking at it now, it is clear to me that I did not fully understand Rust's ownership model and was partially working around that. I was also not clear on how mio (epoll/kqueue) were sending events.

## Ownership Problems

I have a function that, given a token, would find the corresponding connection in the connection slab. It looks like this (and used to be named `find_connection_by_token`):

```rust
fn connection(&mut self, token: Token) -> &mut Connection {
    &mut self.conns[token]
}
```

This function takes `&mut self` because it needs to return a mutable reference to the `Connection`. When I first started writing mob, I did not yet have a good mental model on how to write Rust programs. I fought the borrow checker constantly because I would try to assign the connection to a variable, `let conn = self.connection(token);`, only to have the compiler tell me this mutable reference was preventing me from using the `connection` function again later on in the code. It is now clear to me that I should have structured my code to keep all the connection logic in one place and not try to call `self.connection(token)` from different functions. I was used to working in garbage collected (GC) languages and C, which have no problems if you have multiple mutable references to objects. I also did not have a clear enough mental model of how mio was working in order to design the code to keep the connection logic in one place. In `mio v0.4.x`, you had to implement the `Handler` trait which forced a certain kind of design on the [code](https://github.com/carllerche/mio/blob/v0.4.1/test/test_echo_server.rs#L238).

I did not want to rewrite large parts of mob to remove the connection state though. To make some progress in the short-term, I made sure my code never kept a reference to a connection object. To do this, I made sure to always chain calls when using the connection object. Something like this:

```rust
self.connection(token).reregister(poll)?;
```

This allows me to call `self.connection()` all over my code without hold on to that reference and causing ownership problems. I still think it is a good idea to refactor the code to separate the details of mio and the domain logic of mob, but that is for another time.


## How Events Are Triggered

It is helpful to understand the difference between _level-triggered_ and _edge-triggered_ events before reading the below explanations. Mob receives edge-triggered events. If you are not clear what _level-triggered_ vs _edge-triggered_ means, I suggest you read the section in [mio::Poll](https://docs.rs/mio/0.6.14/mio/struct.Poll.html#edge-triggered-and-level-triggered) that discusses helps define these two terms. If you want even more detail, I suggest reading [Epoll is fundamentally broken 1/2](https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/) and [Epoll is fundamentally broken 2/2](https://idea.popcount.org/2017-03-20-epoll-is-fundamentally-broken-22/). Despite the sensational titles, the author of these blog posts goes through multiple examples of different types of triggering. All three of these links focus on read events. I was fuzzy about how a _write_ event is triggered. I originally thought that it had something to do with the client making a call to read. My current understanding is that the events are triggered based on the changing state of the read and write kernel buffers for that connection (or socket).

The kernel read buffer starts out empty. At some point the kernel receives some data from the client, the kernel writes that data to the read buffer and a read event is triggered. The read buffer is now in a state of non-empty. If the kernel receives more data and writes it to the read buffer while it is in a state of non-empty, another event will not be triggered. Another read event will be triggered if, and only if, the kernel read buffer is in a state of empty and then data is written to it. If mob does not read all of the data, then the subsequent call to `poll` will appear to hang. What is happening is that `poll` will not receive another read event because the kernel read buffer is still in a non-empty state. This is why it is critically important than whenever mob receives a read event that it reads until it receives `WouldBlock`. This ensures the kernel read buffer is put back into a empty state and thus able to trigger another read event if it receives more data.

Write events are a little different because we usually do not have enough data to fill up the kernel write buffer until we receive `WouldBlock`. The kernel write buffer starts out in an empty state. When a connection registers for write, it will receive a write event due to the empty state of the buffer. The connection can then write some data, but in the mob case will most certainly not fill up the buffer as mob messages are quite small. The kernel write buffer is in a non-empty state and will not trigger another write event until the write buffer is empty. The kernel will then try to send the data to the client and once all the data is sent the kernel will trigger another write event (assuming the connection is still registered to receive write events). During the time between the initial write and the kernel sending the contents of the buffer, the connection is still allowed to write until it receives `WouldBlock`.

To round out my understanding, let us also briefly talk about the hangup (hup) event. The hup event works like read and write events. The connection is in an _established_ state. When the client closes their end of the connection, the state changes to closed (or reset) and the connection will receive the hup event.

## Solution

With my improved understanding of ownership and a more accurate mental model of how the kernel sends events, the fix is pretty simple. Before processing any events, make sure the connection is in the slab. The diff of the change is [here](https://github.com/hjr3/mob/pull/23/commits/485487217ddde7d316d7c7b0ac9057696278bc43#diff-4ce93534efc34e923ce01e975eb7ed80R105). Most of the changes are removing code, so let me walk through the important parts.

```rust
if self.token != token && self.conns.contains(token) == false {
    debug!("Failed to find connection for {:?}", token);
    return;
}
```

The `self.token` is the token for the server and is not present in the slab. The slab is indexed by the tokens, so `self.conns.contains()` is a constant time lookup. Much better than iterating through a list of connections. I am not quite done though. If the connection encounters some error, I need to remove it from the slab. To do this I replaced `self.find_connection_by_token(token).mark_reset();` with `self.remove_token(token);`.

In hindsight, this was a pretty obvious change to make. Some [comments](https://www.reddit.com/r/rust/comments/3q0hjt/managing_connection_state_with_mio_herman_j/cwb7n3r/) in the [reddit post](https://www.reddit.com/r/rust/comments/3q0hjt/managing_connection_state_with_mio_herman_j/) on managing the connection state were trying to explain this to me, but I did not get it at the time. There are a few other things in mob that this same pattern of an original naive solution where I now see an obvious improvement to make. I hope to make those changes as well to continue to _diff_ my mindset between 2015 and now.

[post]: /2015/10/23/managing-connection-state-with-mio-rust.html
