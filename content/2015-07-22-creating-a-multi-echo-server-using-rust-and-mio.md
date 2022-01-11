+++
title = "Creating A Multi-echo Server using Rust and mio"
path = "/2015/07/22/creating-a-multi-echo-server-using-rust-and-mio.html"

[taxonomies]
tags=["rustlang", "mio"]
+++

This is my second blog post in a series about async IO. You may want to read [first blog post][first blog post] if you are not familar with mio or epoll/kqueue implementations.

## Basic Setup

At the time of this writing, I am using the newly released mio `0.4.x`. Until recently if you got mio from crates.io, then you will get `0.3.x`. There are breaking changes between these two releases.

I have a [complete working example][complete working example] that has a lot of comments in the source code. I am going to skip over a lot of detail and try to focus on handling a read event and then writing it to all connected clients. If I am making too large of a leap, open up the source to get some more context.


Our example will contain two main parts:

   1. A Server that handles events from our event loop and manages all connecitons.
   1. A Connection that represents new client connections.

The code does not use _unwrap_. I want to properly handle errors to get a feel for something written in mio that is closer to production ready. An error related to a Connection should reset that connection and never tear down the entire server. An error from the server, except during init, should cause a safe shutdown.

## Server

Here is a quick overview of the `Server` struct.

```rust
struct Server {
    // Listening socket for our server.
    sock: TcpListener,

    // We keep track of server token here instead of doing `const SERVER = Token(0)`.
    token: Token,
    
    // A list of connections _accepted_ by our server. This commonly referred to as the
    // _connection slab_.
    conns: Slab<Connection>,

}
```

Our `Server` object will receive all the events from the event loop by implementing `mio::Handler`. A read event for the server token means a new client connection is coming in. We need to _accept_ that new request, create a new `Connection` and add that connection object to our slab. A read event for any other token means we should already have established that connection. We need to forward the read event to that established connection.

```rust
impl Handler for Server {
    fn ready(&mut self, event_loop: &mut EventLoop<Server>, token: Token, events: EventSet) {
        if events.is_readable() {
            if self.token == token {
                self.accept(event_loop);
            } else {

                self.readable(event_loop, token)
                    .and_then(|_| self.find_connection_by_token(token).reregister(event_loop))
                    .unwrap_or_else(|e| {
                        warn!("Read event failed for {:?}: {:?}", token, e);
                        self.reset_connection(event_loop, token);
                    });
            }
        }
    }
}
```

Our accept function will add a new connection to the connection slab. [Slab][slab] is described as a _Slab allocator for Rust_. I just recently [discovered][slab allocator tweet] where the term _slab allocator_ came from. From what I have read about `Slab`, it allows us to use custom types as the index for an vector-like data structure. Within mio, the `Slab` type has been reexported as `pub type Slab<T> = ::slab::Slab<T, ::Token>;`. This means that the `Token` type will be the index and our `Connection` will be the value. Do not get confused, like I was, between the `Slab` type in the _slab_ crate and the `Slab` type mio is reexporting.

Also, I will be using the `Server#find_connection_by_token` method all over the place. It is really just a thin wrapper to look up a connection with a given token: `self.conns[token]`.

Let us see the slab allocator in action:

```rust
    fn accept(&mut self, event_loop: &mut EventLoop<Server>) {

        let sock = // ... skip some boilerplate about accepting a new socket connection

        // `Slab#insert_with` is a wrapper around `Slab#insert`. I like `#insert_with`
        // because I make the `Token` required for creating a new connection.
        //
        // `Slab#insert` returns the index where the connection was inserted.
        // Remember that in mio, the Slab is actually defined as 
        // `pub type Slab<T> = ::slab::Slab<T, ::Token>;`. Token is just a
        // tuple struct around `usize` and Token implemented `::slab::Index`
        // trait. So, every insert into the connection slab will return a new
        // token needed to register with the event loop. Fancy...
        match self.conns.insert_with(|token| {
            debug!("registering {:?} with event loop", token);
            Connection::new(sock, token)
        }) {
            Some(token) => {
                // If we successfully insert, then register our connection.
                match self.find_connection_by_token(token).register(event_loop) {
                    Ok(_) => {},
                    Err(e) => {
                        error!("Failed to register {:?} connection with event loop, {:?}", token, e);
                        self.conns.remove(token);
                    }
                }
            },
            None => {
                // If we fail to insert, `conn` will go out of scope and be dropped.
                error!("Failed to insert connection into slab");
            }
        };

        // We are using edge-triggered polling. Even our SERVER token needs to reregister.
        self.reregister(event_loop);
    }
```

Established connections are forwarded to `Server#readable`. Connections are identified by the token provided to us from the event loop. Once a read has finished, push the receive buffer into the all the existing connections so we can echo it back to all the connections (remember, this is a multi-echo server).

```rust
    fn readable(&mut self, event_loop: &mut EventLoop<Server>, token: Token) -> io::Result<()> {
        debug!("server conn readable; token={:?}", token);
        let message = try!(self.find_connection_by_token(token).readable());

        if message.remaining() == message.capacity() { // is_empty
            return Ok(());
        }

        let mut bad_tokens = Vec::new();

        // Queue up a write for all connected clients.
        for conn in self.conns.iter_mut() {
            let conn_send_buf = ByteBuf::from_slice(message.bytes());
            conn.send_message(conn_send_buf)
                .and_then(|_| conn.reregister(event_loop))
                .unwrap_or_else(|e| {
                    error!("Failed to queue message for {:?}: {:?}", conn.token, e);
                    // We have a mutable borrow for the connection, so we cannot 
                    // remove until the loop is finished
                    bad_tokens.push(conn.token)
                });
        }

        for t in bad_tokens {
            self.reset_connection(event_loop, t);
        }

        Ok(())
    }
```

## Connection

The `Connection` object represents a client connection. This looks similar to `Server`, with a few differences. I need keep track of what events we are interested in. By default, the connection is always interested in a read event. Only when we push messages into the `send_queue` will the connection be interested in a write event.

```rust
struct Connection {
    // handle to the accepted socket
    sock: TcpStream,

    // token used to register with the event loop
    token: Token,

    // set of events we are interested in
    interest: EventSet,

    // messages waiting to be sent out
    send_queue: Vec<ByteBuf>,
}
```

We are using `MutByteBuf` to read data from the socket. MutByteBuf, part of the [bytes crate][bytes crate], is a heap allocated slice that mio supports internally. I prefer to use this as it does the work of tracking how much of our slice has been used. I chose a capacity of 2048 after reading [some mio source code][some mio source code] as that seems like a good size of streaming. If you are wondering what the difference between messaged based and continuous streaming read the answer to this [StackOverflow question][StackOverflow question]. TLDR: UDP vs TCP. We are using TCP.

```rust
    fn readable(&mut self) -> io::Result<ByteBuf> {

        let mut recv_buf = ByteBuf::mut_with_capacity(2048);

        // we are PollOpt::edge() and PollOpt::oneshot(), so we _must_ drain
        // the entire socket receive buffer, otherwise the server will hang.
        loop {
            match self.sock.try_read_buf(&mut recv_buf) {
                // the socket receive buffer is empty, so let's move on
                // try_read_buf internally handles WouldBLock here too
                Ok(None) => {
                    debug!("CONN : we read 0 bytes");
                    break;
                },
                Ok(Some(n)) => {
                    debug!("CONN : we read {} bytes", n);

                    // if we read less than capacity, then we know the
                    // socket is empty and we should stop reading. if we
                    // read to full capacity, we need to keep reading so we
                    // can drain the socket. if the client sent exactly capacity,
                    // we will match the arm above. the recieve buffer will be
                    // full, so extra bytes are being dropped on the floor. to
                    // properly handle this, i would need to push the data into
                    // a growable Vec<u8>.
                    if n < recv_buf.capacity() {
                        break;
                    }
                },
                Err(e) => {
                    error!("Failed to read buffer for token {:?}, error: {}", self.token, e);
                    return Err(e);
                }
            }
        }

        // change our type from MutByteBuf to ByteBuf so we can use it to
        // write
        Ok(recv_buf.flip())
    }
```

The result of the read is pushed into all the existing connections write queue by `Server#readble` (we went over this function above). The last thing to do is to then write this message back to the client. The `try_write_buf` method is similar to the `try_read_buf` method we used above except that it expects a `ByteBuf`. I chose to only write one buffer from the queue to the client per write event. If there are still buffers in the queue, we remainig interested in writable events. If queue is empty, then we are no longer interested in write events.

```rust
    fn writable(&mut self) -> io::Result<()> {

        try!(self.send_queue.pop()
            .ok_or(Error::new(ErrorKind::Other, "Could not pop send queue"))
            .and_then(|mut buf| {
                match self.sock.try_write_buf(&mut buf) {
                    Ok(None) => {
                        debug!("client flushing buf; WouldBlock");

                        // put message back into the queue so we can try again
                        self.send_queue.push(buf);
                        Ok(())
                    },
                    Ok(Some(n)) => {
                        debug!("CONN : we wrote {} bytes", n);
                        Ok(())
                    },
                    Err(e) => {
                        error!("Failed to send buffer for {:?}, error: {}", self.token, e);
                        Err(e)
                    }
                }
            })
        );

        if self.send_queue.is_empty() {
            self.interest.remove(EventSet::writable());
        }

        Ok(())
    }
```

I am just getting into async io and mio, so my implementation may not be ideal, but it works. We have a functioning multi-echo server that is resistant to errors. The source also contains a simple client that will repeatedly write a message to the server and then read a message.

One thing that this code does not do well is handle reads from a client. In order to do that well, we need to establish a simple _protocol_. I am working through that now and will go over that in my [next post][protocol-blog-post].

[first blog post]: /2015/07/12/my-basic-understanding-of-mio-and-async-io.html
[complete working example]: https://github.com/hjr3/mob/blob/multi-echo-blog-post/src/main.rs
[slab]: https://crates.io/crates/slab
[slab allocator tweet]: https://twitter.com/hermanradtke/status/622863648273215488
[bytes crate]: https://crates.io/crates/bytes
[some mio source code]: https://github.com/carllerche/mio/blob/eed4855c627892b88f7ca68d3283cbc708a1c2b3/src/io.rs#L23-27
[StackOverflow question]: http://stackoverflow.com/questions/3017633/difference-between-message-oriented-protocols-and-stream-oriented-protocols
[protocol-blog-post]: /2015/09/12/creating-a-simple-protocol-when-using-rust-and-mio.html
