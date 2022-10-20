+++
title = "Sending UDP Messages in Node.js Without DNS Lookups"

[taxonomies]
tags=["node", "udp"]
+++

At work, I recently inherited a node service that was sending hundreds of metrics to DataDog using the [brightcove/hot-shots](https://github.com/brightcove/hot-shots) StatsD client. I noticed the node service was making a very large number of calls to `dns.lookup` because we were specifying a domain name when sending UDP messages. While working towards a solution to this problem, I noticed other people had run into this same issue but there was no one sharing what a good solution looked like. I present a general solution that worked for me in hopes that it will be useful to you.

<!-- more -->

In a hurry? You can [skip](#final) to the final program.

## Simple Example

Let us start with a simplified example. We will create a UDP socket and send some messages. Let us not want to bother with creating a server to listen for our messages. We can send them to `www.example.com`. Nothing will be listening for these messages at `www.example.com`, but UDP is fire and forget so it will not matter.

```js
import dgram from 'node:dgram';
import { Buffer } from 'node:buffer';
import util from 'util';

const host = 'www.example.com';

const socket = dgram.createSocket('udp4');
socket.send2 = util.promisify(socket.send);

const msg = Buffer.from('foo');

await socket.send2(msg, 0, msg.length, 8125, host);
console.log('msg 1 sent');
await socket.send2(msg, 0, msg.length, 8125, host);
console.log('msg 2 sent');

socket.close();
```

Note: We promisify our `socket.send` function for two reasons:

1. Using `await` makes the flow of our code sequential.
1. We want to call `socket.close` after all messages are sent

If we had a server that was sending StatsD messages using UDP, we almost certainly would not want to block on (`await`) each call to `socket.send`.

We run our code and get the following:

```
$ node udp.mjs
msg 1 sent
msg 2 sent
```

### Avoid Using Domain Names with UDP

We prefer UDP for sending data like metrics because it is fast. We do not want the overhead of TCP and we are fine dropping some connections. However, `www.example.com` is a domain name. From the [socket.send](https://nodejs.org/docs/latest-v18.x/api/dgram.html#socketsendmsg-offset-length-port-address-callback) node docs:

> If the value of address is a host name, DNS will be used to resolve the address of the host.

A few sentences later, the docs warn us:

> DNS lookups delay the time to send for at least one tick of the Node.js event loop.

Depending on how fast our DNS server is, we may be delayed for much longer than one tick of the event loop. However, things actually get worse. In [Implementation considerations](https://nodejs.org/api/dns.html#dnslookup), we are warned:

> Though the call to dns.lookup() will be asynchronous from JavaScript's perspective, it is implemented as a synchronous call to getaddrinfo(3) that runs on libuv's threadpool. This can have surprising negative performance implications for some applications, see the UV_THREADPOOL_SIZE documentation for more information.

The `getaddrinfo` function is written in C. It is a blocking function, which would cause problems for our event loop. To prevent blocking, the call to `getaddrinfo` is made using an internal threadpool. From [UV_THREADPOOL_SIZE](https://nodejs.org/api/cli.html#uv_threadpool_sizesize):

> Because libuv's threadpool has a fixed size, it means that if for whatever reason any of these APIs takes a long time, other (seemingly unrelated) APIs that run in libuv's threadpool will experience degraded performance.

If we are sending a lot of UDP messages, we absolutely do not want to be using `dns.lookup`.

Let us prove this out by wrapping the `dns.lookup` function:

```js
import dns from 'node:dns';

const originalLookup = dns.lookup;
dns.lookup = (...args) => {
  console.log('called dns.lookup');
  originalLookup(...args);
};
```

Now, when we run our example, we see:

```
$ node udp.mjs
called dns.lookup
called dns.lookup
msg 1 sent
called dns.lookup
msg 2 sent
```

Each time we want to send a message over UDP, we have to first use DNS to look up the IP address. We also notice that we are making three calls to `dns.lookup` when we only want to send two messages, but let us come back to that.

## Using IP Address Instead of Domain Name

To avoid repeated calls to `dns.lookup`, we can get the IP address once at the top of our program and then pass the IP address instead to `socket.send`.

Look up the IP address at the very top of our program. We do this prior to wrapping the `dns.lookup` code.

```js
import { lookup } from 'node:dns/promises';
const ip = await lookup(host);
```

Use the IP address instead of the domain name.

```
await socket.send2(msg, 0, msg.length, 8125, ip.address);
```

Our code now looks like this:

```js
import dgram from 'node:dgram';
import { Buffer } from 'node:buffer';
import util from 'util';

import dns from 'node:dns';
import { lookup } from 'node:dns/promises';

const originalLookup = dns.lookup;
dns.lookup = (...args) => {
  console.log('called dns.lookup');
  originalLookup(...args);
};

const host = 'www.example.com';
const ip = await lookup(host);

const socket = dgram.createSocket('udp4');
socket.send2 = util.promisify(socket.send);

const msg = Buffer.from('foo');

await socket.send2(msg, 0, msg.length, 8125, ip.address);
console.log('msg 1 sent');
await socket.send2(msg, 0, msg.length, 8125, ip.address);
console.log('msg 2 sent');

socket.close();
```

Now, we expect to bypass all calls to `dns.lookup` when we run our code.

```
$ node udp.mjs
called dns.lookup
called dns.lookup
msg 1 sent
called dns.lookup
msg 2 sent
```

This is surprising and we are not the only ones who think so. This behavior was first called out in [nodejs/node#35130](https://github.com/nodejs/node/issues/35130) but was dismissed with a "won't fix" response. It was brought again in [nodejs/node#39468](https://github.com/nodejs/node/issues/39468) because (as the docs said above), we are still delayed by at least one tick of the event loop as shown in [b3723fac05](https://github.com/nodejs/node/blob/b3723fac05aa86a4e0604e218dbd8ae24609172b/lib/dns.js#L155-L164).

## Avoiding `dns.lookup` When Using IP Address

We want to bypass this behavior so we cans end UDP messages without this unnecessary delay. Globally modifying the `dns.lookup` function is dangerous and not a path we want to go down. Instead, we configure our socket to use a custom lookup function.

```js
const socket = dgram.createSocket({
  type: 'udp4',
  lookup: (hostname, options, callback) => {
    dns.lookup(hostname, options, callback);
  },
});
```

We also want to change our `socket.send` functions to go back to using the hostname. You could still use the IP, but that has a downside we will discuss later.

```
await socket.send2(msg, 0, msg.length, 8125, host);
```

We have not changed any real behavior yet. Let us confirm our code is still working.

```js
$ node udp.mjs
called dns.lookup
called dns.lookup
msg 1 sent
called dns.lookup
msg 2 sent
```

Now, we can remove the call to `dns.lookup` and use our IP address. We reference the [dns.lookup](https://nodejs.org/api/dns.html#dnslookuphostname-options-callback) docs to see how we use the `callback` function.

```diff
-    dns.lookup(hostname, options, callback);
+    callback(null, ip.address, ip.family);
```

Now, we expect to not see any calls to the `dns.lookup` function when we run our code.

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

Oh, dear. We seem to have broken something at a low level. Let us do some quick debugging and add the following as the first line in our custom lookup function:

```js
console.log('called lookup', { hostname, options });
```

When we run our code again, we will see this line:

```
calling dns.lookup { hostname: '0.0.0.0' }
```

Our socket is first trying to bind to a local address (in this case `0.0.0.0`). Our custom lookup function returned `93.184.216.34` instead. The socket cannot bind to a non-local address like `93.184.216.34` and emitted an error that told us as much. Now that we know that our lookup function can be called in unexpected ways, let us change the function to bypass `dns.lookup` only when `hostname` matches our expected domain name.

```js
const socket = dgram.createSocket({
  type: 'udp4',
  lookup: (hostname, options, callback) => {
    if (hostname === host) {
      callback(null, ip.address, ip.family);
      return;
    }

    console.log('calling dns.lookup', { hostname });
    dns.lookup(hostname, options, callback);
  },
});
```

Let us also send a message to different domain name so we can really see this working in action.

```
await socket.send2(msg, 0, msg.length, 8125, 'example.net');
console.log('msg 3 sent');
```

Now, we run our code and it is working as expected:

```
$ node udp.mjs
calling dns.lookup { hostname: '0.0.0.0' }
msg 1 sent
msg 2 sent
calling dns.lookup { hostname: 'example.net' }
msg 3 sent
```

We handled the initial lookup to `0.0.0.0` and the lookup for `example.net`. We could explicitly check for `0.0.0.0` using `net.isIP` and bypass `dns.lookup` there as well. It only happens when the socket is initially created, so it probably is not a big deal either way.

## Honoring TTL When Caching DNS Records

We are now successfully caching the IP address and avoiding an extra tick of the event loop when sending messages. However, we have failed to account for the DNS record changing. Remember when we made sure to use the domain name instead of the IP address in the call to `socket.send`? Honoring the TTL of the domain name is why.
 When we look up an IP address using DNS, the response includes a time to live (TTL) value that determines how long the answer is good for. We should update our code to honor this value. However, `dns.lookup` does not provide the TTL in the response. In fact, `dns.lookup` is handling a lot of complexity for us. We are going to have to switch to `dns.resolve` (or [dns.resolve4](https://nodejs.org/api/dns.html#dnsresolve4hostname-options-callback) in our case) to explicitly talk to DNS. This has the following changes:

- we will now get a TTL back
- we may now get back more than one IP address
- we cannot pass an IP address to `dns.resolve`
- we cannot pass non-domain host names, such as `localhost`, to `dns.resolve`

We swap out `dns.lookup` and use `dns.resolve4` instead. We chose `dns.resolve4` instead of `dns.resolve` because we are using IPv4 only. The interface is pretty similar to `dns.lookup` except that we need to explicitly ask for the TTL and we have to deal with an array of results.

```js
import { resolve4 } from 'node:dns/promises';

const host = 'www.example.com';
const ipAddresses = await resolve4(host, { ttl: true });

const { address, ttl } = ipAddresses.pop();
```

We choose the first result and then store those values in some variables we will use as a basic cache.

```
let cacheIpAddr = address;
let cacheTtl = ttl * 1000; // store ttl in ms
let cacheTimestamp = Date.now();
```

When our lookup functions is called, we will first check to see if the cache is expired. If the cache is expired, we will make another call to `dns.resolve4` and update the IP address and the TTL. It is important we update the TTL each time because the TTL is a _live_ value. For example, the DNS configuration for `www.example.com` may specify a TTL of 300 seconds but when we ask DNS for the IP address there are only 187 seconds left. We want to avoid the mistake of assuming the 187 seconds is the TTL for subsequent requests.

```js
const socket = dgram.createSocket({
  type: 'udp4',
  lookup: (hostname, options, callback) => {
    if (hostname === host) {
      const now = Date.now();
      if (
        (now - cacheTimestamp > cacheTtl)
      ) {
        dns.resolve4(hostname, { ttl: true }, (err, addresses) => {
          if (err) {
            console.error(err);
            return;
          }

          cacheIpAddr = addresses[0].address;
          cacheTtl = addresses[0].ttl * 1000;

          console.log('cache updated', { cacheIpAddr, cacheTtl });

          callback(null, address, 4);
        });
      } else {
        console.log('using cached address');
        callback(null, address, 4);
      }

      return;
    }

    console.log('calling dns.lookup', { hostname });
    dns.lookup(hostname, options, callback);
  },
});
```

And we can add in another UDP message that will only be sent after the cache expires.

```js
const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));
// note: www.example.com has a _very_ long ttl
await delay(cacheTtl);
await socket.send2(msg, 0, msg.length, 8125, host);
console.log('msg 4 sent');
```


```
$ node udp.mjs
calling dns.lookup { hostname: '0.0.0.0' }
using cached address
msg 1 sent
using cached address
msg 2 sent
calling dns.lookup { hostname: 'example.net' }
msg 3 sent
cache updated { cacheIpAddr: '93.184.216.34', cacheTtl: 81501000 }
msg 4 sent
```


One big downside to this approach is that it is susceptible to a [cache stampede](https://en.wikipedia.org/wiki/Cache_stampede). Let us assume our service sends out a StatsD metric every 10 ms and the `dns.resolve4` function takes 100 ms. When the cache expires, we may have ~10 functions that are trying to send metrics cache miss and use `dns.resolve4`. As the rate of metrics increases, we may overwhelm our DNS server with a burst of requests every 15 minutes. A better solution would be to use a slightly stale IP address while we make _one_ call to `dns.resolve4`.

## Avoiding A Cache Stampede

We can introduce a lock to prevent the stampede. If the lock is not set and the cache is expired, then we will lazily update the cache. This means there is a small amount of time where cached IP address _may_ not match the actual IP address(es) in DNS. Losing seconds of metrics is an acceptable trade-off to avoid the cache stampede.

```diff
 let cacheTimestamp = Date.now();
+let resolveLock = false;
 const socket = dgram.createSocket({
   type: 'udp4',
   lookup: (hostname, options, callback) => {
     if (hostname === host) {
       const now = Date.now();
       if (
+        resolveLock === false &&
         (now - cacheTimestamp > cacheTtl)
       ) {
+        resolveLock = true;
+        console.log('lazily refreshing ip address...');
         dns.resolve4(hostname, { ttl: true }, (err, addresses) => {
+          resolveLock = false;
           if (err) {
             console.error(err);
             return;
@@ -33,13 +38,11 @@ const socket = dgram.createSocket({

           console.log('cache updated', { cacheIpAddr, cacheTtl });

-          callback(null, address, 4);
+          // intentionally do not call the callback
         });
-      } else {
-        console.log('using cached address');
-        callback(null, address, 4);
       }

+      callback(null, address, 4);
       return;
     }
```
<a name="final"></a>
Our final program now looks like this:

```js
import dgram from 'node:dgram';
import { Buffer } from 'node:buffer';
import util from 'util';

import dns from 'dns';
import { resolve4 } from 'node:dns/promises';

const host = 'www.example.com';
// note: resolve4 will not work with IP addresses nor will it work with values like 'localhost'
const ipAddresses = await resolve4(host, { ttl: true });

const { address, ttl } = ipAddresses.pop();

let cacheIpAddr = address;
let cacheTtl = ttl * 1000; // store ttl in ms
let cacheTimestamp = Date.now();
let resolveLock = false;
const socket = dgram.createSocket({
  type: 'udp4',
  lookup: (hostname, options, callback) => {
    if (hostname === host) {
      const now = Date.now();
      if (
        resolveLock === false &&
        (now - cacheTimestamp > cacheTtl)
      ) {
        resolveLock = true;
        console.log('lazily refreshing ip address...');
        dns.resolve4(hostname, { ttl: true }, (err, addresses) => {
          resolveLock = false;
          if (err) {
            console.error(err);
            return;
          }

          cacheIpAddr = addresses[0].address;
          cacheTtl = addresses[0].ttl * 1000;

          console.log('cache updated', { cacheIpAddr, cacheTtl });

          // intentionally do not call the callback
        });
      }

      callback(null, address, 4);
      return;
    }

    console.log('calling dns.lookup', { hostname });
    dns.lookup(hostname, options, callback);
  },
});
socket.send2 = util.promisify(socket.send);

const msg = Buffer.from('foo');

await socket.send2(msg, 0, msg.length, 8125, host);
console.log('msg 1 sent');
await socket.send2(msg, 0, msg.length, 8125, host);
console.log('msg 2 sent');
await socket.send2(msg, 0, msg.length, 8125, 'example.net');
console.log('msg 3 sent');

const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));
// note: www.example.com has a _very_ long ttl
await delay(cacheTtl);
await socket.send2(msg, 0, msg.length, 8125, host);
console.log('msg 4 sent');

socket.close();
```

We run it and see that it now update the cache in the background without blocking the UDP message being sent.

```
$ node udp.mjs
calling dns.lookup { hostname: '0.0.0.0' }
msg 1 sent
msg 2 sent
calling dns.lookup { hostname: 'example.net' }
msg 3 sent
lazily refreshing ip address...
msg 4 sent
cache updated { cacheIpAddr: '93.184.216.34', cacheTtl: 81501000 }
```

## Preventing DNS Lookup in hot-shots StatsD Client

We can apply this same approach to the hot-shots StatsD client. A recent patch made it possible to pass UDP socket options when creating the client. Since [a399dda](https://github.com/brightcove/hot-shots/commit/a399dda99fb1bf2b15e53646b3ef5d8cbb0b90c9) landed in `v9.2.0` you can do:

```
const client = new StatsD({
  host,
  port,
  udpSocketOptions: {
    type: 'udp4',
    lookup: (hostname, options, callback) => {
      // code our program above
    },
  },
});
```

## Source Code

You can get the full source code at [hjr3/upd-no-dns](https://github.com/hjr3/udp-no-dns). This code used Node.js v18. There is an `.nvmrc` file in the repo that contains the exact version.
