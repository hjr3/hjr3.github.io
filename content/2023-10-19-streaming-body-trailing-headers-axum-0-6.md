+++
title = "Stream a Body With Trailers in axum 0.6"

[taxonomies]
tags=["rustlang", "hyper", "axum"]
+++

Hyper is designed to support streaming bodies. The current version of axum, v0.6, supports streaming a response. If we want to include [trailers](https://datatracker.ietf.org/doc/html/rfc7230#section-4.4) (sometimes called "trailing headers") then we need to implement our own custom body.

<!-- more -->

Caveats:

- The custom body implementation only works in axum 0.6, which uses http-body 0.4.4. The http-body crate changed in v1.0.0-rc.2. The concept is the same, but the custom `StreamBody` type will be different.
- Trailers are only supported in hyper using HTTP/2. You can monitor https://github.com/hyperium/hyper/issues/2719 for HTTP/1.1 support.

If you want to send trailer headers in HTTP/1.1 or you do not want to implement your own `Body`, please refer to [Stream a Body With Trailers in hyper 1.0 and axum 0.7](/streaming-body-trailing-headers-hyper-1-0-and-axum-0-7/)

### Set up

In order to send trailers, we need an axum server that uses HTTP/2. Also, most implementations of HTTP/2 require TLS. Let us start from [axum/examples/tls-rustlls](https://github.com/tokio-rs/axum/tree/1e5be5bb693f825ece664518f3aa6794f03bfec6/examples/tls-rustls). This will give us a working HTTP/2 server that uses self-signed TLS certificates.

We need to make a few changes to the `Cargo.toml` in order for the example to work:

```diff
 [package]
-name = "example-tls-rustls"
+name = "axum-trailers"
 version = "0.1.0"
 edition = "2021"
 publish = false

 [dependencies]
-axum = { path = "../../axum" }
+axum = { version = "0.6.20", features = ["http2"] }
 axum-server = { version = "0.3", features = ["tls-rustls"] }
 ```

We can now verify our server working:

```
$ cargo run
   Compiling axum-trailers v0.1.0 (/Users/herman/Code/axum-trailers)
    Finished dev [unoptimized + debuginfo] target(s) in 3.65s
     Running `target/debug/axum-trailers`
```

```
$ curl -k https://localhost:3000
Hello, World!%
```

### Streaming Body

Before sending trailers, we need to change our `handler` function to stream a response. First, add `tokio-stream` as a dependency:

```
$ cargo add tokio-stream
```

We then need to modify our imports:

```diff
 use axum::{
+    body::StreamBody,
     extract::Host,
     handler::HandlerWithoutStateExt,
     http::{StatusCode, Uri},
+    response::IntoResponse,
     response::Redirect,
+    response::Response,
     routing::get,
     BoxError, Router,
 };
 use axum_server::tls_rustls::RustlsConfig;
-use std::{net::SocketAddr, path::PathBuf};
+use std::{convert::Infallible, net::SocketAddr, path::PathBuf};
+use tokio::sync::mpsc;
 use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
 ```

Finally, we can replace the existing handler with one that streams a body:

```rust
async fn handler() -> impl IntoResponse {
   let (tx, rx) = mpsc::channel::<Result<String, Infallible>>(2);

   tokio::spawn(async move {
       tx.send(Ok("hello...".to_string())).await.unwrap();
       tokio::time::sleep(std::time::Duration::from_secs(2)).await;
       tx.send(Ok("world".to_string())).await.unwrap();
   });

   let stream = tokio_stream::wrappers::ReceiverStream::new(rx);
   let body = StreamBody::new(stream);

   Response::builder()
       .status(StatusCode::OK)
       .body(body)
       .unwrap()
}
```

We spawn a _task_ that will send `hello...`, wait 2 seconds and then send `world`. Hyper knows how to correctly process a stream, but does not know what do with the _receiver_ from the `mpsc::channel`. We use `tokio-stream` to convert the receiver into a stream and use that as our response body.

Note: HTTP/2 does not use a `Transfer-Encoding` header. You can add one, but hyper will properly strip it out.

We can test that our response is now streaming a body using curl.

```
$ curl -k --no-buffer https://localhost:3000/
hello...world%
```

With the `--no-buffer` flag, you should notice a pause between `hello...` and `world`.

### Sending Trailers

In http-body v0.4.4, the [Body](https://github.com/hyperium/http-body/blob/a97da649b6dc93660931fc6f0bdb6aa2db64e50d/src/lib.rs#L56-L62) trait has a `poll_trailers` method handles the sending of trailers at the end of the body. In axum v0.6, [StreamBody](https://github.com/tokio-rs/axum/blob/1e5be5bb693f825ece664518f3aa6794f03bfec6/axum/src/body/stream_body.rs) always returns `None`:

```rust
fn poll_trailers(
    self: Pin<&mut Self>,
    _cx: &mut Context<'_>,
) -> Poll<Result<Option<HeaderMap>, Self::Error>> {
    Poll::Ready(Ok(None))
}
```

#### Custom `StreamBody`

We can start from axum's `StreamBody` implementation and add support for trailers.

Copy the [StreamBody](https://github.com/tokio-rs/axum/blob/1e5be5bb693f825ece664518f3aa6794f03bfec6/axum/src/body/stream_body.rs) implementation from axum to our server:

```
curl --silent "https://raw.githubusercontent.com/tokio-rs/axum/1e5be5bb693f825ece664518f3aa6794f03bfec6/axum/src/body/stream_body.rs" --output src/stream_body.rs
```

We need to make some changes to the import statments in `src/stream_body.rs`:

1. Rename `crate` to `axum`
1. Remove `use http::HeaderMap` as axum re-exports this dependency
1. Add `http::HeaderMap` to the existing `use axum { ... }` import.

```diff
-use crate::{
+use axum::{
     body::{self, Bytes, HttpBody},
+    http::HeaderMap,
     response::{IntoResponse, Response},
     BoxError, Error,
 };
     ready,
     stream::{self, TryStream},
 };
-use http::HeaderMap;
 use pin_project_lite::pin_project;
 use std::{
     fmt,
 ```

We then modify the `StreamBody` struct to include `trailers`. This will allow us to store the trailers in our response.

```diff
     pub struct StreamBody<S> {
         #[pin]
         stream: SyncWrapper<S>,
+        trailers: Option<HeaderMap>,
     }
 }
 ```

We also need to set `trailers` to `None` when creating a new stream:

```diff
    pub fn new(stream: S) -> Self
    where
        S: TryStream + Send + 'static,
        S::Ok: Into<Bytes>,
        S::Error: Into<BoxError>,
     {
         Self {
             stream: SyncWrapper::new(stream),
+            trailers: None,
         }
     }
 }

 impl<S> IntoResponse for StreamBody<S>
 ```

Add a `set_trailers` method to `StreamBody` so we can add trailer headers from our response:

```diff
+    pub fn set_trailers(&mut self, headers: HeaderMap) {
+        self.trailers = Some(headers);
+    }
```

Finally, modify `poll_trailers` to send any headers we set:

```diff
    fn poll_trailers(
        self: Pin<&mut Self>,
        _cx: &mut Context<'_>,
    ) -> Poll<Result<Option<HeaderMap>, Self::Error>> {
-        Poll::Ready(Ok(None)
+        Poll::Ready(Ok(self.project().trailers.take()))
    }
```

#### Update Response

Now that we have a `StreamBody` implementaiton that will send headers, we can update `handler` in `src/main.rs` to include trailers.

Update the imports to use the `StreamBody` we just created:

```diff
+mod stream_body;
+
 use axum::{
-    body::StreamBody,
     extract::Host,
     handler::HandlerWithoutStateExt,
     http::{StatusCode, Uri},
@@ -20,6 +21,8 @@ use std::{convert::Infallible, net::SocketAddr, path::PathBuf};
 use tokio::sync::mpsc;
 use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

+use crate::stream_body::StreamBody;
```

We modify our response to include a header:

```diff
     let stream = tokio_stream::wrappers::ReceiverStream::new(rx);
-    let body = StreamBody::new(stream);
+    let mut body = StreamBody::new(stream);
+    let mut headers = axum::http::HeaderMap::new();
+    headers.insert("chunky-trailer", "foo".parse().unwrap());
+
+    body.set_trailers(headers);

     Response::builder()
         .status(StatusCode::OK)
+        .header("Trailers", "chunky-trailer")
         .body(body)
         .unwrap()
```

Note: we must include a `Trailers` header that names the trailer headers we want to send.

We can use curl to verify that our trailer header is sent. Note that we must include the verbose flag, `-v`, in order to see the headers.

```
$ curl -v -k --no-buffer https://localhost:3000/
...snip
> GET / HTTP/2
> Host: localhost:3000
> user-agent: curl/7.79.1
> accept: */*
>

< HTTP/2 200
< trailers: chunky-trailer
< date: Thu, 19 Oct 2023 22:28:06 GMT
<
hello...world< chunky-trailer: foo
* Connection #0 to host localhost left intact
```

Note: the `< chunky-trailer: foo` is on the same line as `hello...world` because we did not buffer the body.

You can find the complete source code at [https://github.com/hjr3/axum-trailers](https://github.com/hjr3/axum-trailers)
