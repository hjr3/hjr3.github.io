+++
title = "Developer Experience: Fast Startup Is Not The Only Speed Metric"

[taxonomies]
tags=["dx"]
+++

I was reading [How fast is your shell?](https://registerspill.thorstenball.com/p/how-fast-is-your-shell) and it got me thinking about the difference types of speed developers prioritize. A lot of articles and advice focus on initial startup, which can be thought of as _acceleration_. We should also consider, what happens after we are done accelerating, how easy it is to maintain our velocity.

<!-- more -->

Look, I am biased towards fast startup times. Here is my zsh startup time:

```
$ time zsh -i -c exit
zsh -i -c exit  0.04s user 0.02s system 94% cpu 0.063 total
$ time zsh -i -c exit
zsh -i -c exit  0.04s user 0.02s system 95% cpu 0.066 total
$ time zsh -i -c exit
zsh -i -c exit  0.04s user 0.02s system 93% cpu 0.067 total
$ time zsh -i -c exit
zsh -i -c exit  0.04s user 0.02s system 94% cpu 0.069 total
$ time zsh -i -c exit
zsh -i -c exit  0.04s user 0.02s system 93% cpu 0.065 total
```

I am open and close my my shell quite often. As a result, my shell configuration is fairly minimal. I generally omit features, like custom completions, to achieve this. This is not the only way people work.

When I pair with some of my colleagues, I observe that they have a different style of working. They open VS Code and then open a single shell within the VS Code UI. They usually leave VS Code open all day and use that one shell. The initial startup time of VS Code and the the shell within VS Code is not that important to them. What is important to them is how easily they can maintain their productivity level of the course of the day.

People that have a long-loved shell, are probably better off adding additional functionality, such as custom completions. These additional features help maintain their productivity velocity. My suggestions to them about improving their shell startup time will often seem unimportantto them. They are optimizing for a different outcome than I am.

A while back, I switched from vim to neovim. Prior to the switch, I kept my vim configuration pretty minimal to avoid slowing down vim's startup time. I sacraficed some of the more advanced, heavier plugins as a result. I was fine with this trade-off. Neovim makes it much easier to lazy-load plugins, which allows me to maintain a fast startup time while also providing me access to features, such as languages servers.

I am not lazy loading anything in zsh. Maybe I should? I found [zsh-lazyload](https://github.com/qoomon/zsh-lazyload) from a quick Google search, but I have not looked into it closer. I know fish has [autoloading function](https://fishshell.com/docs/current/tutorial.html#autoloading-functions) which is basically lazy loading. The downside of lazy loading approaches is the additional complexity. I will only consider lazy loading things in my shell if it is relatively straight-forward and a first-class concern of the shell.

I like the [How fast is your shell?](https://registerspill.thorstenball.com/p/how-fast-is-your-shell) article because it matches my own development style. I also try to keep in mind that others have styles of working that require a different set of optimizations. Ideally, we can be creative enough to get fast startup times and access to all the features we want using concepts like lazy-loading.
