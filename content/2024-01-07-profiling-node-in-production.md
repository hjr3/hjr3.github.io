+++
title = "Profiling Node.js in Production"

[taxonomies]
tags=["node", "v8", "profiling"]
+++

At work, I lead a team responsible for a Node.js service that serves a lot of GraphQL queries. We recently noticed some servers in the cluster were running much slower than others. We had used [0x][0x] in the past to profile Node.js services locally. In this case, we could not identify the problem locally and needed a solution to profile Node.js in production to identify the cause of the slowdown.

<!-- more -->

To profile in production, I wanted to expose a route that would enable profiling for a short amount of time and then give us access to the results. Working with another engineer, we decided on the following requirements:
- the route would only be accessible through an internal port because our application was internet facing
- the data would be returned as part of the route when the profiling was finished

<details>
  <summary>Node and package versions used in this article</summary>

  - node - 18.12.0
  -  express - 4.18.2
  -  v8-profiler-next - 1.10.0

</details>

## Setup

Let us create a simple express app that represents the Node.js service we want to profile.

```js
import express from "express";

const app = express();

app.get("/", (req, res) => {
  res.send("hello world");
});

app.listen(8000);
```

## Internally Enable Profiling at Runtime

I use `node --prof /path/to/main.js` when profiling locally. I use [0x][0x] which calls the application using the `--prof` flag and automatically generates a flamegraph. The problem is that we want to profile a running service in production for a few seconds.

We found two packages on npm that can enable profiling at runtime
- [hyj1991/v8-profiler-next](https://github.com/hyj1991/v8-profiler-next)
- [node-inspector/v8-profiler](https://github.com/node-inspector/v8-profiler)

After reading [https://github.com/node-inspector/v8-profiler/issues/137](https://github.com/node-inspector/v8-profiler/issues/137), we chose `v8-profiler-next` because our Node.js is running node v20.

Our production Node.js service is internet facing. We only want our new profiling route available on our internal VPN. Our example express app listens for requests on port `8000`, which we can pretend is our public port. We create a separate express app listening on port `8001`.

```js
import v8Profiler from "v8-profiler-next";

v8Profiler.setGenerateType(1);
const mgmt = express();

mgmt.get("/profile", (req, res) => {
  const timeoutMs = req.query.timeout || 1000;

  const title = `myapp-${Date.now()}.cpuprofile`;
  v8Profiler.startProfiling(title);
  setTimeout(() => {
    const profile = v8Profiler.stopProfiling(title);
    profile.export(function (error, result) {
      res.attachment(title);
      res.send(result);
      profile.delete();
    });
  }, timeoutMs); 
});

mgmt.listen(8001);
```

<details>
  <summary>Entire main.js file</summary>

```js
import express from "express";
import v8Profiler from "v8-profiler-next";

const app = express();

app.get("/", (req, res) => {
  res.send("hello world");
});

app.listen(8000);

v8Profiler.setGenerateType(1);
const mgmt = express();

mgmt.get("/profile", (req, res) => {
  const timeoutMs = req.query.timeout || 1000;

  const title = `myapp-${Date.now()}.cpuprofile`;
  v8Profiler.startProfiling(title);
  setTimeout(() => {
    const profile = v8Profiler.stopProfiling(title);
    profile.export(function (error, result) {
      res.attachment(title);
      res.send(result);
      profile.delete();
    });
  }, timeoutMs); 
});

mgmt.listen(8001);
```

</details>

We set `v8Profiler.setGenerateType(1);` to use the new _tree_ profiling format. Most modern tooling that analyzes CPU profiles prefer this format.

The `/profile` route will enable profiling and then return the output. We allow a `timeout` query parameter to control how long the profiling would run. We want to minimize the amount of time we profile for two reasons:
- profiling slows the service down
- the output can get quite large

Our production service receives a lot of traffic, so a few seconds of profile output was more than enough to start analyzing.

We give each profile a name so we do not get them confused. Our example app uses `myapp-${Date.now()}.cpuprofile`. You may also consider including the host name in the filename if available. The `.cpuprofile`  extension is the convention for profiling output.

After the profile is complete, the profile output is sent back in the response. We use `res.attachment` to add the [Content-Disposition](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition) header to the response to indicate the content should be downloaded and saved locally.

## Profiling the Service

We first start our Node.js application using `node main.js`.

We can call our application public routes on port `8000`

```shell
$ curl localhost:8000/
hello world%
```

We can profile our application using port `8001`

```
$ curl localhost:8001/profile --remote-name --remote-header-name --silent
$ ls myapp-1704639385130.cpuprofile
myapp-1704639385130.cpuprofile
```

- The `--remote-name` option instructs `curl` to save that data into a local file instead of writing to `stdoout`.
- The `--remote-header-name` option instructs `curl` to use the `Content-Disposition` filename instead of extracting a filename from the URL.
- The `--silent` option instructs `curl` to disable the progress meter. This is my personal preference and not required.
## Analyzing the CPU Profile Using Flamegraphs

We spent some time searching for the best way to view the results as a [flamegraph](https://www.brendangregg.com/flamegraphs.html). I was used to [0x][0x] handling this by default. I liked the output of [0x][0x] but I did not want to hack the code to render the results for an external file.

A lot of the suggestions I read said to use Chrome's `Profile` tab. Newer versions of Chrome no longer have this tab and I could not get the `Performance Insights` tab render my `.cpuprofile` files. Other suggestions were to manually convert the file into an svg image. A svg file is fine, but I wanted something a little better.

I tried [thlorenz/flamegraph](https://github.com/thlorenz/flamegraph) but I received an error when I tried to use it and gave up after a few minutes.

I happened to stumble upon [jlfwong/speedscope](https://github.com/jlfwong/speedscope) which was exactly what I was looking for. It is easy to internally install and use. If you are working on open source you can use [https://www.speedscope.app/](https://www.speedscope.app/).

## Conclusion

Once we were able to view the `.cpuprofile` flamegraph's we quickly identified that our  [Apollo Server - Subscription](https://www.apollographql.com/docs/apollo-server/data/subscriptions) implementation for was the culprit. We were using [davidyaha/graphql-redis-subscriptions](https://github.com/davidyaha/graphql-redis-subscriptions) to load balance subscriptions across the cluster and it was using a lot of CPU, possibly due to the use of [async generators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncGenerator) which have had known performance issues. We are using node v20 in production and supposedly those performance are fixed.

The issue was that when a particular server handled too many subscriptions, then it slowed way down. We are still investigating and changed implementations in the meantime. The cluster is now performing much better.

## Addendum
### startProfiling Options

I noticed the `startProfiling` function accepts three parameters:

```typescript
export function startProfiling(name?: string, recsamples?: boolean, mode?: 0 | 1): void;
```

I could not find any documentation on what the `recsamples` and `mode` options did though. The default value for `recsamples` is `true` and for `mode` is 0.

I dug through the code and eventually found the answer to `recsamples` in [v8-profiler.h](https://github.com/v8/v8/blob/10.1.10/include/v8-profiler.h#L387-L388) which says
> |record_samples| parameter controls whether individual samples should be recorded in addition to the aggregated tree

I also found the answer to `mode` in [cpu-profiler.cc](https://github.com/hyj1991/v8-profiler-next/blob/ba0b6b9c46b6469466da5e995b7cb4099de1a5c1/src/cpu_profiler/cpu_profiler.cc#L68-L75) which toggle for eager vs lazy logging.

[0x]: https://github.com/davidmarkclements/0x
