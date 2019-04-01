---
layout: post
title: "fastly/lucet in five minutes"
tags:
- rustlang
status: publish
type: post
published: true
---

Lucet is Fastly's [native WebAssembly compiler and runtime][lucet]. I am a big fan of Rust, Fastly and WASM. Especially WASM on the server via [WASI][WASI]. I jumped right in and tried to get my own lucet program running, but the [setup][README] is a rather long process. My plan was to introduce lucet to some colleagues at my local Rust meetup. I am a huge fan of Rust, but the compile times are an issue. Spending 30 minutes on setup was a non-starter. I was excited when I saw that Fastly published a [Docker container][fastly/lucet docker]:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">The Fastly Lucet image is now available on the Docker Hub. `docker pull fastly/lucet` and you’re all set.</p>&mdash; Frank Denis (@jedisct1) <a href="https://twitter.com/jedisct1/status/1111330113864548353?ref_src=twsrc%5Etfw">March 28, 2019</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

However, this container was built for people developing on lucet. The container has all of the required dependencies, but still requires the same initial setup process. So, I decided to take advantage of Docker's multi-stage build process to create a container that has lucet already built. It comes in at the slim size of 107 MB, which should make it fast to download.

Here is how you can get lucet running a simple Hello World program in 5 minutes.

```bash
$ docker pull hjr3/lucet
$ cat hello.c
#include <stdio.h>

int main(void)
{
    puts("Hello world");
    return 0;
}
$ docker run --rm -it -v "$(pwd)":/usr/local/src hjr3/lucet wasm32-unknown-wasi-clang -Ofast -o hello.wasm hello.c
$ docker run --rm -it -v "$(pwd)":/usr/local/src hjr3/lucet lucetc-wasi -o hello.so hello.wasm
$ docker run --rm -it -v "$(pwd)":/usr/local/src hjr3/lucet lucet-wasi hello.so
Hello world
```

This docker container provides a version of clang capable of compiling to wasm via the `wasm32-unknown-wasi-clang` command. It is a not a requirement to compile your program to wasi using `wasm32-unknown-wasi-clang` in this docker container. The only requirement is that you compile the program to wasi before using `lucetc-wasi`. Also, take note that `lucetc-wasi` and `lucet-wasi` have very similar spellings, but are indeed two different programs.

~~If you are wondering why I did not demo converting a Rust program to WASI, we are blocked until [wasm32-unknown-wasi][wasi PR] is a valid target in rustup. As soon as that target is available, then I plan on creating another post showing how to get Rust + lucet working together.~~ See [WASI example using Rust and Lucet][Rust example] for a Rust example that runs on lucet.

Lucet is not 1.0 yet and I expect to be changing it a lot. As of this moment, the [hjr3/lucet][hjr3/lucet container] container is built against [fastly/lucet commit e6b399b][e6b399b]. As new changes come in, I will do my best to update the container. I may setup an automated process if this proves useful to people. I will tag each version against the fastly/lucet commit the container is built against.

[lucet]: https://www.fastly.com/blog/announcing-lucet-fastly-native-webassembly-compiler-runtime
[WASI]: https://hacks.mozilla.org/2019/03/standardizing-wasi-a-webassembly-system-interface/
[README]: https://github.com/fastly/lucet/blob/8632b16faf2353727c9aa272d3ac65885eb9e1b9/README.md
[fastly/lucet docker]: https://hub.docker.com/r/fastly/lucet
[wasi PR]: https://github.com/rust-lang/rust/pull/59464
[hjr3/lucet container]: https://hub.docker.com/r/hjr3/lucet
[e6b399b]: https://github.com/fastly/lucet/commit/e6b399b3fc6794f8f78a8bf6ad404ca640a090c4
[Rust example]: /2019/04/01/wasi-example-using-rust-and-lucet.html
