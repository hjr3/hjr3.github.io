---
layout: post
title: "Initial v0.1.0 release of weldr - a reverse proxy written in Rust"
tags:
- rustlang
status: publish
type: post
published: true
redirect_from: /2017/02/15/alacrity-reverse-proxy-initial-release-rust.html
---

Note: This project was originally named [alacrity](https://github.com/hjr3/alacrity/issues/69).

Over the past few months I have been working on a building a reverse proxy, called [weldr](https://github.com/hjr3/weldr), in Rust. I have just published the initial [release](https://github.com/hjr3/weldr/releases/tag/0.1.0) of weldr. I have been interested in doing something with networks in Rust. I have spent a lot of time building hypermedia APIs, so doing something with HTTP seemed like a good fit. I started out using mio and later switched to [tokio](https://github.com/tokio-rs/tokio). While this was fun to do, I quickly realized that I was spending most of my time implementing the HTTP spec instead of building the features I was most excited about. The [hyper](https://github.com/hyperium/hyper) HTTP library was also switching over to tokio and way ahead of where I was at. After talking with Sean, I made the decision to use hyper as the foundation for weldr moving forward. There are still a number of things a proxy must do in order to conform with the various HTTP related RFCs. I will be working on those proxy specific requirements while adding the features that I want in a reverse proxy.

There are two general problems I am trying to solve with weldr. The first problem is that popular open source proxies do not work as well as I would like them to in dynamic cloud/container environments. The reason is that dynamic parts of a proxy, such as the list of backend servers, are defined by a configuration file that is read when the proxy is started. I want to build a proxy that has a minimal configuration file and drive most of the behavior through a set of APIs. This may make it harder to use weldr for simple use cases, but I hope weldr can make more complex environments a lot easier. I will note that there are some products that do this now, such as NGINX Plus, but those are cost prohibitive for many.

The second, more aspirational, problem is around the _availability_ of the proxy. If I want to put a reverse proxy in front of a critical cluster of web servers, I have two basic options: use an active/passive setup or use DNS. For an active/passive setup, the go to is [keepalived](http://www.keepalived.org/). I think keepalived does a great job, but it is a real pain to setup and ensure it is working correctly. Even more so if you are trying to automate the creation of servers. I want to start mulitple weldr proxy servers and have them automatically determine a leader with one or more followers. This means that keepalived kind of logic must be embedded inside weldr. I plan on using Raft to accomplish this. The current Rust raft library is missing some features that I require though. I hope to contribute those at some point in the future. The other solution, DNS, is even more challenging to get working correctly. It may be necessary though in order to handle a large number of requests. I have some very early thoughts about making this work better as well. I hope to write about those thoughts more in the future.

Weldr is definitely not production ready, but it is working well enough that you can play with it. I would love any feedback on what I have currently done so far and my plans for the future.
