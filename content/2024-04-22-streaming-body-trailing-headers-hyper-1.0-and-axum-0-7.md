+++
title = "Stream a Body With Trailers in hyper 1.0 and axum 0.7"

[taxonomies]
tags=["rustlang", "hyper", "axum"]
+++

Hyper supports sending HTTP/1.1 Chunked Trailer Fields as of [v1.1.0](https://github.com/hyperium/hyper/releases/tag/v1.1.0). The [http-body](https://crates.io/crates/http-body) is now at [v1.0](https://docs.rs/http-body/1.0.0/http_body/) as well and uses frames to allow a stream to return data and trailers.

<!-- more -->

We were able to write trailer fields in [Stream a Body With Trailers in axum 0.6](/streaming-body-trailing-headers-axum-0-6/) by using HTTP/2 and writing our own custom body implementation. The hyper ecosystem has expanded trailer support which makes it easier to send trailer fields on both HTTP/1.1 and HTTP/2 without writing a custom body.

## Set up

We start with the basic [Hello, World!](https://hyper.rs/guides/1/server/hello-world/) server from the hyper guide. As of this writing, the guide basically implements this [example](https://github.com/hyperium/hyper/blob/226305d0fc78ab780aa5a1084e013a3b0a39e4d8/examples/hello.rs).

Cargo.toml

```toml
[package]
name = "hyper-send-trailers"
version = "0.1.0"
edition = "2021"

[dependencies]
hyper = { version = "1", features = ["full"] }
tokio = { version = "1", features = ["full"] }
http-body-util = "0.1"
hyper-util = { version = "0.1", features = ["full"] }
```

src/main.rs

```rust
use std::convert::Infallible;
use std::net::SocketAddr;

use http_body_util::Full;
use hyper::body::Bytes;
use hyper::server::conn::http1;
use hyper::service::service_fn;
use hyper::{Request, Response};
use hyper_util::rt::TokioIo;
use tokio::net::TcpListener;

async fn hello(_: Request<hyper::body::Incoming>) -> Result<Response<Full<Bytes>>, Infallible> {
    Ok(Response::new(Full::new(Bytes::from("Hello, World!"))))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));

    let listener = TcpListener::bind(addr).await?;

    loop {
        let (stream, _) = listener.accept().await?;

        let io = TokioIo::new(stream);

        tokio::task::spawn(async move {
            if let Err(err) = http1::Builder::new()
                .serve_connection(io, service_fn(hello))
                .await
            {
                eprintln!("Error serving connection: {:?}", err);
            }
        });
    }
}
```

We can now verify our server working:

```
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.12s
     Running `target/debug/hyper-send-trailers
```

```
$ curl --raw http://localhost:3000/
Hello, World!%
```

## Send Trailer As Part of a Stream

We will now replace the `hello` handler with one that is both a streaming response and that includes a trailer field after the stream is finished.

First, add tokio-stream as a dependency:

```
$ cargo add tokio-stream
```

We then need to modify our imports:

```diff
-use http_body_util::Full;
-use hyper::body::Bytes;
+use http_body_util::StreamBody;
+use hyper::body::{Bytes, Frame};
 use hyper::server::conn::http1;
 use hyper::service::service_fn;
-use hyper::{Request, Response};
+use hyper::{HeaderMap, Request, Response, StatusCode};
 use hyper_util::rt::TokioIo;
 use tokio::net::TcpListener;
+use tokio::sync::mpsc;
+use tokio_stream::wrappers::ReceiverStream;
```

We are replacing the `Full` body implementation with the `StreamBody` implementation. We also need to import the `Frame` type from the body to inform hyper what kind of streaming data is being sent. Last, we import `tokio::sync::mpsc` to send data across tasks and `tokio_stream::wrappers::ReceiverStream` in order to stream the output received from the task.

Our handler will now look like this:

```rust
type Data = Result<Frame<Bytes>, Infallible>;
type ResponseBody = StreamBody<ReceiverStream<Data>>;
async fn hello(_: Request<hyper::body::Incoming>) -> Result<Response<ResponseBody>, Infallible> {
    let (tx, rx) = mpsc::channel::<Data>(2);

    // some async task
    tokio::spawn(async move {
        // some expensive operations
        tx.send(Ok(Frame::data(Bytes::from("hello..."))))
            .await
            .unwrap();

        tokio::time::sleep(std::time::Duration::from_secs(2)).await;
        tx.send(Ok(Frame::data(Bytes::from("world"))))
            .await
            .unwrap();

        // headers based off expensive operation
        let mut headers = HeaderMap::new();
        headers.insert("chunky-trailer", "foo".parse().unwrap());
        tx.send(Ok(Frame::trailers(headers))).await.unwrap();
    });

    let stream = ReceiverStream::new(rx);
    let body = StreamBody::new(stream);

    Ok(Response::builder()
        .status(StatusCode::OK)
        .header("Trailer", "chunky-trailer") // trailers must be declared
        .body(body)
        .unwrap())
}
```

The types get complicated, so I chose to use some type aliases to make it easier to understand. The `Data` type is the data being produced by our task and will be sent over the `mpsc::channel`. The `ResponseBody` type is our stream of `Data`.

We create a bounded `mpsc::channel` to send data from a task we will spawn to the response. We spawn a task, which represents any data we want to stream. For this example, I am sending `hello...`, sleeping for 2 seconds and then sending `world`. We use `Frame::data` to tell hyper that these bytes are the chunked body. Once we are finished sending data, we can include trailer fields. I am sending back a header map containing a single trailer. We use `Frame::trailers` to tell hyper that these headers are the trailer fields.

The `tokio_stream` crate allows us to convert the receiving side of the `mpsc::channel` into a stream. We then create a `StreamBody`, which implements the `Body` trait hyper requires, from the receiving stream.

Finally, we build our response. Hyper strictly follows the HTTP/1.1 spec and will only include chunked trailer fields that are specfied in the `Trailer` response header.We can use curl to verify that our trailer header is sent.

That is it! We can use curl to verify that our trailer header is sent:

```
$ curl --raw -H "TE: trailers" http://localhost:3000/
8
hello...
5
world
0
chunky-trailer: foo
```

We use the `--raw` flag to see the individual chunks and trailer fields returned from our server. The `TE: trailers` header is how the client informs the server that it is willing to recieve headers and is required in order for hyper to send the trailer fields.


You can find the complete source code at [https://github.com/hjr3/axum-trailers/tree/hyper-send-trailers](https://github.com/hjr3/axum-trailers/tree/hyper-send-trailers)

## Using axum and IntoResponse

Using axum provides us with a really nice quality of life improvement: `IntoResponse`. This trait allows us to omit a lot of the complicated types that start showing up when we have complex `Body` implementations and are using futures.

We can change our `hello` handler from

```rust
async fn hello(_: Request<hyper::body::Incoming>) -> Result<Response<ResponseBody>, Infallible>
```

to

```rust
async fn hello() -> impl IntoResponse
```

which is much easier to work with, especially when we start dealing with futures.

## Trailer Field Without Streams

Suppose we want to send a trailer field without a stream. We can wrap our `Body` implementation with an adapter that allows us to send trailer fields using a future. I am going to use axum for this example to avoid any complex types.

We need to import `BodyExt` from the `http_body_util` crate. This trait allows us to call `with_trailers` on a type that implements `Body`.

```diff
-use http_body_util::StreamBody;
+use http_body_util::{BodyExt, StreamBody};
```

Next, we can use `with_trailers` to create a future that returns a header map:

```rust
let body = body.with_trailers(async move {
    let mut headers = HeaderMap::new();
    headers.insert("trailer2", "bar".parse().unwrap());
    Some(Ok(headers))
});
```

Finally, remember to update the `Trailer` field to specify the `trailer2` header:

```diff
.status(StatusCode::OK)
-.header("Trailer", "chunky-trailer")
+.header("Trailer", "chunky-trailer, trailer2")
.body(body)
```

When we send a client request, we will get back both headers:

- one as part of our stream
- a second one that comes from a separate future

```
$ curl --raw -H "TE: trailers" http://localhost:3000/
8
hello...
5
world
0
chunky-trailer: foo
trailer2: bar
```

You can find the complete source code at [https://github.com/hjr3/axum-trailers/tree/axum-0-7](https://github.com/hjr3/axum-trailers/tree/axum-0-7)
