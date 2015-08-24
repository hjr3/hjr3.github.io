---
layout: post
title: "Using Docker to Test Rust on Linux"
tags:
- rustlang
status: publish
type: post
published: true
---

I use a MacBook Air as my main laptop. I have been using Vagrant to test rust programs on linux. This feels a little heavy to me though as I have to create a Vagrant machine for each repository. The up and halt phases of the Vagrant box are a little slow and each machine eats away at my available hard drive space. I do not need an entire virutalized operating system, just a place to test my programs. This seems like a good use case for [Docker][Docker].

The first thing I had to do was get [boot2docker][boot2docker] installed. This was pretty straight-forward, but did take a fair bit of time. The second thing was finding an upstream Rust container (I think this is called an image) to use. I do not want to build one myself. Doing a [search on Docker Hub][search on Docker Hub] I chose the [schickling/rust][schickling/rust] container. The repo info had a simple walkthrough, the Dockerfile itself seemed straightforward and it included gdb.

This container is setup to be used interatively or to run commands. To run cargo tests:

```bash
$ docker run --rm -it -v $(pwd):/source schickling/rust cargo test
```

Let us go over the command. If you are familiar with Docker, you can skip this section. To start, `docker run` runs a command in a new contianer. The `--rm`, `-it` and `-v` flags are very common when running Docker containers. Here is what they mean:

   * `--rm` - automatically remove the container when it exists. This means when the command is over, the container will stop running. This removes the _container_, but not the `schickling/rust` _image_.
   * `-it` - make the docker container interactive and allocate a tty. This basically means your shell will work.
   * `-v` - mount a volume. In the example above, it binds the local _present working directory_ to the `/source` directory inside the container.

After the flags, we specify the upstream Docker container name `schickling/rust` and then finally our command.

If you want to run experiment inside the linux container, just omit a command. The docker file used to build`schickling/rust` specifies `bash` as the default command.

```bash
$ docker run --rm -it -v $(pwd):/source schickling/rust cargo test
```

This is basically like a `vagrant ssh` command. You will be given a shell inside the container. I do this when playing with [mob][mob] because I want to run the server, then the client in various ways. Example:

```bash
$ docker run --rm -it -v $(pwd):/source schickling/rust
root@f237a067addb:/source# cargo build
root@f237a067addb:/source# RUST_LOG=trace ./target/debug/mob-server 2> server.log &
root@f237a067addb:/source# ./target/debug/mob-client
root@f237a067addb:/source# fg
```

Make sure you run `cargo build` inside the container as Linux cannot run the executable built on OS X. If you see the error `bash: ./target/debug/mob-server: cannot execute binary file` then you need to `cargo build`. I then run the mob server in the background and send the log output to a file. I run the mob client (sometimes repeatedly). When done, I use the `fg` command to bring the mob server process back into the foreground where I can terminate it (using Ctrl-C). I then exit the container (using Ctrl-D), the container is cleaned up and I can start fresh again if I want.

Docker was fairly easy to get setup and I have found it to be more efficient for these types of use-cases. The hardest part was getting [boot2docker][boot2docker] installed correctly. Docker will not completely replace Vagrant on my machine, but it certainly has found a place.

[Docker]: https://www.docker.com/
[boot2docker]: http://boot2docker.io/
[search on Docker Hub]: https://hub.docker.com/search/?q=rust&page=1&isAutomated=0&isOfficial=0&starCount=0&pullCount=0
[schickling/rust]: https://hub.docker.com/r/schickling/rust/
[mob]: https://github.com/hjr3/mob
