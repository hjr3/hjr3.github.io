+++
title = "Send UDP Messages in Node.js Without dns.lookup"
aliases=["/sending-udp-messages-in-nodejs-without-dns-lookups"]

[taxonomies]
tags=["node", "udp"]
+++

At work, I recently inherited a node service that was sending metrics to DataDog using the [brightcove/hot-shots](https://github.com/brightcove/hot-shots) StatsD client. While investigating some issues with `dns.lookup`, I noticed other people had run into this same issue but there was no one sharing what a solution might look like.

<!-- more -->

_Note: This post was significantly edited on November 6, 2022._

In a hurry? You can [skip](#preventing-dns-lookup-in-hot-shots-statsd-client) to the solution.

## `dns.lookup` Is Always Called

Let us create a simple program to send a message via UDP. We _can_ use a domain name with `node:dgram`, but it is bad idea. I explain why [here](#addendum-avoid-domain-names). Let us assume we have a single IP address instead.

```js
import dgram from 'node:dgram';
import dns from 'node:dns';

const originalLookup = dns.lookup;
dns.lookup = (...args) => {
  console.log('called dns.lookup');
  originalLookup(...args);
};

const ip = '93.184.216.34'; // www.example.com
const socket = dgram.createSocket('udp4');

socket.send('foo', 8125, ip, (err) => {
  socket.close();
});
```

When we run this program, we expect to bypass all calls to `dns.lookup` when we run our code.

```
$ node udp.mjs
called dns.lookup
called dns.lookup
```

This is surprising and we are not the only ones who think so. This behavior was first called out in [nodejs/node#35130](https://github.com/nodejs/node/issues/35130) but was dismissed with a _won't fix_ response. It was brought up again in [nodejs/node#39468](https://github.com/nodejs/node/issues/39468) because (as the docs said above), we are still delayed by at least one tick of the event loop as shown in [b3723fac05](https://github.com/nodejs/node/blob/b3723fac05aa86a4e0604e218dbd8ae24609172b/lib/dns.js#L155-L164).

## Avoiding `dns.lookup` When Using IP Address

To avoid `dns.lookup`, we configure our socket to use a custom lookup function.

```js
const socket = dgram.createSocket({
  type: 'udp4',
  lookup: (hostname, _options, callback) => {
    callback(null, hostname, 'IPv4');
  },
});
```

The `hostname` value will be the value of `ip`. Now, when we run it we will not see any calls made to `dns.lookup`.

```
$ node udp.mjs
```

### The Value of `hostname` Is Not Always What We Expect

We might consider swapping out `hostname` for `ip` in the callback, but that will cause a problem.

```diff
-   callback(null, hostname, 'IPv4');
+   callback(null, ip, 'IPv4');
```

```
$ node udp.mjs
node:internal/errors:484
    ErrorCaptureStackTrace(err);
    ^

Error: bind EADDRNOTAVAIL 93.184.216.34
    at node:dgram:359:20
    at lookup (file:///Users/herman/Code/udp-no-dns/udp.mjs:14:5)
    at UDP.lookup4 (node:internal/dgram:24:10)
    at Socket.bind (node:dgram:325:16)
    at Socket.send (node:dgram:645:10)
    at node:internal/util:364:7
    at new Promise (<anonymous>)
    at Socket.send2 (node:internal/util:350:12)
    at file:///Users/herman/Code/udp-no-dns/udp.mjs:21:14
Emitted 'error' event on Socket instance at:
    at node:dgram:361:14
    at lookup (file:///Users/herman/Code/udp-no-dns/udp.mjs:14:5)
    [... lines matching original stack trace ...]
    at file:///Users/herman/Code/udp-no-dns/udp.mjs:21:14 {
  errno: -49,
  code: 'EADDRNOTAVAIL',
  syscall: 'bind',
  address: '93.184.216.34'
}
```

The issue is that our `socket.send` first tries to bind to a local address (e.g. `0.0.0.0`), which calls our custom lookup function. This is why our first example printed _called dns.lookup_ twice: first for the local address and the second time for the `host` parameter of `socket.send`. Our custom lookup function returned `93.184.216.34` both times. The socket cannot bind to a non-local address like `93.184.216.34` and emitted an error that told us as much. Now that we know that our lookup function can be called in unexpected ways, let us change the function to bypass `dns.lookup` only when `hostname` matches our expected domain name.

If we want to be really safe, we can consider calling `dns.lookup` for any value of `hostname` other than `ip`.

```js
const socket = dgram.createSocket({
  type: 'udp4',
  lookup: (hostname, options, callback) => {
    if (hostname === ip) {
      callback(null, ip, 'IPv4');
      return;
    }

    dns.lookup(hostname, options, callback);
  },
});
```

## Preventing DNS Lookup in hot-shots StatsD Client

Now that we know about custom lookup functions, we can apply this same approach to the hot-shots StatsD client. A recent patch made it possible to pass UDP socket options when creating the client. Since [a399dda](https://github.com/brightcove/hot-shots/commit/a399dda99fb1bf2b15e53646b3ef5d8cbb0b90c9) landed in `v9.2.0` you can do:

```
const client = new StatsD({
  host,
  port,
  udpSocketOptions: {
    type: 'udp4',
    lookup: (hostname, options, callback) => {
      // our program above
    },
  },
});
```

## Addendum: Avoid Domain Names

We prefer UDP for sending data like metrics because it is fast. We do not want the overhead of TCP and we are fine dropping some connections. When using a domain name, the docs warn us:

> DNS lookups delay the time to send for at least one tick of the Node.js event loop.

Depending on how fast our DNS server is, we may be delayed for much longer than one tick of the event loop. However, things actually get worse. In [Implementation considerations](https://nodejs.org/api/dns.html#dnslookup), we are warned:

> Though the call to dns.lookup() will be asynchronous from JavaScript's perspective, it is implemented as a synchronous call to getaddrinfo(3) that runs on libuv's threadpool. This can have surprising negative performance implications for some applications, see the UV_THREADPOOL_SIZE documentation for more information.

The `getaddrinfo` function is written in C. It is a blocking function, which would cause problems for our event loop. To prevent blocking, the call to `getaddrinfo` is made using an internal threadpool. From [UV_THREADPOOL_SIZE](https://nodejs.org/api/cli.html#uv_threadpool_sizesize):

> Because libuv's threadpool has a fixed size, it means that if for whatever reason any of these APIs takes a long time, other (seemingly unrelated) APIs that run in libuv's threadpool will experience degraded performance.

If we are sending a lot of UDP messages, we absolutely do not want to be using domain names.

### DNS Caching

We may be forced to use a domain name if the IP address (or addresses) change. In that case, our best bet is to use some sort of DNS cache. Choosing a proper implementation is for another post. However, once we decide on an cache implementation, we can combine the DNS cache with our custom lookup function to avoid calling `dns.lookup`.
