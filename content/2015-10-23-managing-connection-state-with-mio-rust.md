+++
title = "Managing Connection State With mio"
path = "/2015/10/23/managing-connection-state-with-mio-rust.html"

[taxonomies]
tags=["rustlang", "mio"]
+++

I would wager most people who choose mio to solve their async IO problems are expecting more abstraction in the library. The oft repeated question of _Why doesn't mio have callbacks?_ is evidence of this. In fact, it is a design goal of mio to add only as much abstraction necessary to provide a consistent API for the various OS async IO implementations. A consequence of this design decision is that there are subtle behavioral differences between platforms. It may be tempting to let mio manage the state of various connections, but I have found that this can have unintended consequences. For example, until recently [mio internally buffered kqueue events][mio-pull-270]. The way epoll is designed does not warrant buffering events and those of us using kqueue encountered some [interesting panic behavior on kqueue][mio-pull-256]. I was able to avoid this issue when I started managing connection state in conjunction with the recently added `Handler::tick()` function.

The idea for adding `Handler::tick()` came from a discussion abbout [deregister bevhavior][mio-issue-219]. The idea is that at the end of each event loop _tick_, the `Handler::tick()` function will be called. By default this function does nothing. We can implement this function to act as a checkpoint to sync the state of our connections with mio before the start of the next event loop _tick_. We have three types of state:

   * **socket state** - whether the connection is reset or not.
   * **event state** - whether we need to register, reregister or do nothing.
   * **read/write state** - whether we are in the middle of a read/write or not. I discussed a solution to this in my post on [Creating A Simple Protocol When Using Rust and mio][Creating A Simple Protocol When Using Rust and mio].

## Socket State

Our connection socket can stop working for many different reasons. When this does happen, we need to remove the connection from the connection slab. One straight-forward approach is to immediately remove the connection from the slab when there is an error related to the socket. Keep in mind though that just because we had an error when trying to handle an event does not mean that there is not another event for that same token. If we try to handle that later event by looking up that connection using a token, we will inadvertently panic. We are now forced to try and keep track of whether a token still exists in the slab inside our `Handler`.

Instead of removing the connection from the slab immediately, we can keep track of whether the connection is reset or not inside the `Connection` struct. If we encounter an error, we will mark the socket as reset and leave the connection in the connection slab until the event loop tick is finished. Now that we have this information local to the connection, our `Handler` can check whether or not the connection is reset before trying to dispatch events to it. Finally, when our `Handler::tick()` method is called, we can check each connection to see if it is reset. If the connection is reset, we can then remove the connection from the slab. Since we did this at the end of the event loop, we can now be confident there are no more spurious events for our token.

Let us implement a simple way to keep track of socket state. The first thing we need to do is add an `is_reset: bool` variable to our `Connection` struct. If `is_reset` is _true_, then we will remove the connection from our connection slab. We will also create two new functions on our `Connection`:

```rust
impl Connection {
   pub fn mark_reset(&mut self) {
      trace!("connection mark_reset; token={:?}", self.token);

      self.is_reset = true;
   }

   #[inline]
   pub fn is_reset(&self) -> bool {
      self.is_reset
   }
}
```

Now the server can quickly determine if a connection has already been reset. If a connection is reset, we want to drop any _readable_ or _writeable_ events. If the connection is not reset, we are confident that we can dispatch an event to that connection. If there is an error when dispatching the event to the connection, then we want to mark that connection as reset.

```rust
let conn = self.find_connection_by_token(token);

if conn.is_reset() {
   info!("{:?} has already been reset", token);
   return;
}

conn.writable().unwrap_or_else(|e| {
   warn!("Write event failed for {:?}, {:?}", token, e);
   conn.mark_reset();
});
```

At the end of the event loop tick, we can loop through our connections and check if any are reset. If they are, we then remove them from the connection slab. Unfortunately, there is not a real good way to iterate over the slab and remove connections from it. Future changes to the [slab crate][slab crate] should make this easier by adding features like [Vec::retain][Vec::retain].

```rust
fn tick(&mut self, event_loop: &mut EventLoop<Server>) {
   trace!("Handling end of tick");

   let mut reset_tokens = Vec::new();

   for c in self.conns.iter() {
      if c.is_reset() {
         reset_tokens.push(c.token);
      }
   }

   for token in reset_tokens {
      match self.conns.remove(token) {
         Some(_c) => {
            debug!("reset connection; token={:?}", token);
         }
         None => {
            warn!("Unable to remove connection for {:?}", token);
         }
      }
   }
}
```

Notice that we do not call the `EventLoop::deregister()` method when a connection is removed from the slab. When we remove a connection from the slab, mio will internally deregister the connection so no more events will be sent. If we call deregister too early, some async I/O implementations (such as kqueue) will send that event as `Token(0)`.

## Event State

When I started using mio, I put calls to rereregister [all over the place][multi-echo-code]. I found a couple of problems with this approach. The first problem is that it becomes increasingly difficult to keep track of when connections are getting added to or removed from the event loop. The second problem is that any spurious event has a good chance of causing a panic. Remember, this is asynchronous behavior and our mental model is often incorrect. I believe the best strategy is to handle all registration related activities inside of `Handler::tick()`. We can make it a goal not to reregister a connection more than once per event loop tick. We should also make it a goal not to reregister if the connection has not received an event.

Similar to our strategy with tracking socket state, we can add an `is_idle: bool` to our `Connection` struct. We will also add two similar functions:

```rust
impl Connection {
   pub fn mark_idle(&mut self) {
      trace!("connection mark_idle; token={:?}", self.token);

      self.is_idle = true;
   }

   #[inline]
   pub fn is_idle(&self) -> bool {
      self.is_idle
   }
}
```

At the bottom of our `Handler::ready()` method, we need to mark the connection as being idle:

```rust
// self.token is our `Server` token. we do not want to mark that idle
if self.token != token {
   self.find_connection_by_token(token).mark_idle();
}
```

Our `Handler::tick()` method will now need to reregister any connection that is in an idle state. We can add combine the check for reregisration with the check for reset connections in the same loop. We end up with:

```rust
fn tick(&mut self, event_loop: &mut EventLoop<Server>) {
    trace!("Handling end of tick");

    let mut reset_tokens = Vec::new();

    for c in self.conns.iter_mut() {
        if c.is_reset() {
            reset_tokens.push(c.token);
        } else if c.is_idle() {
            c.reregister(event_loop)
                .unwrap_or_else(|e| {
                    warn!("Reregister failed {:?}", e);
                    c.mark_reset();
                    reset_tokens.push(c.token);
                });
        }
    }

    for token in reset_tokens {
        match self.conns.remove(token) {
            Some(_c) => {
                debug!("reset connection; token={:?}", token);
            }
            None => {
                warn!("Unable to remove connection for {:?}", token);
            }
        }
    }
}
```

The full code can be found here: [https://github.com/hjr3/mob/tree/state-blog-post](https://github.com/hjr3/mob/tree/state-blog-post).

We now have four variables tracking various parts of our `Connection` state: `is_reset`, `is_idle`, `read_continuation` and `write_continuation`. The latter two being discussed in a [previous blog post][Creating A Simple Protocol When Using Rust and mio]. There is some overlap amongst these variables and I am thinking about how to represent all this state with one _state_ variable on the `Connection` class.

We are also doing a loop over the connection slab for each event loop tick. This can get heavy if we have a lot of connections in the slab. Usually connections are not dropping off that often and we if we are not under load we may not have many connections eligible to be reregistered. Right now, I am willing to take the perf hit in order to not crash. However, I am thinking about ways to accomplish the safety without having to loop so often.

While the soluations may not be ideal, I think it is worth talking about some of the challenges I faced getting mob working on top of mio. Some of the answers have been organic in nature and I will continue to improve them as I learn more.

[mio-pull-256]: https://github.com/carllerche/mio/pull/265
[mio-pull-270]: https://github.com/carllerche/mio/pull/270
[mio-issue-219]: https://github.com/carllerche/mio/issues/219
[multi-echo-code]: https://github.com/hjr3/mob/blob/multi-echo-blog-post/src/main.rs
[Creating A Simple Protocol When Using Rust and mio]: /2015/09/12/creating-a-simple-protocol-when-using-rust-and-mio.html
[slab crate]: https://github.com/carllerche/slab
[Vec::retain]: https://doc.rust-lang.org/nightly/collections/vec/struct.Vec.html#method.retain
