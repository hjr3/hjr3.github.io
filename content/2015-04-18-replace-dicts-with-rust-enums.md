+++
title = "Replace Dicts With Rust Enums"
path = "/2015/04/18/replace-dicts-with-rust-enums.html"

[taxonomies]
tags=["rustlang"]
+++

I listened to the [Changelog podcast on Rust](https://thechangelog.com/151/) recently and loved the remark about enabling web developers to get into systems programming. I have been thinking about the way we design code in the web world versus the way we design code in Rust. In the web world, we often use a hash/dict/map to hide hard-coded values behind a nicer interface. Consider an example where you would want to write a function to create the escape sequence for colors in a TTY terminal. You might write something like this in Ruby:

```ruby
def color(fg, bg=:default)
    fg_codes = {
      :black => 30,
      :red => 31,
      :green => 32,
      :yellow => 33,
      :blue => 34,
      :magenta => 35,
      :cyan => 36,
      :white => 37,
      :default => 39,
    }
    bg_codes = {
      :black => 40,
      :red => 41,
      :green => 42,
      :yellow => 43,
      :blue => 44,
      :magenta => 45,
      :cyan => 46,
      :white => 47,
      :default => 49,
    }
    fg_code = fg_codes.fetch(fg)
    bg_code = bg_codes.fetch(bg)
    escape "#{fg_code};#{bg_code}m"
  end
```

The foreground and the background color codes are accessed by passing in the names of the color as a symbol via `color(:black, :white)`. If you tried to port this code directly into Rust, you would run into a few problems. First, there are no symbols in Rust. You might think to work around this by using strings instead of symbols. Then you start looking at the [documentation for a HashMap](http://doc.rust-lang.org/std/collections/struct.HashMap.html) and realize maps are quite a bit more involved than in Ruby. After fighting with the syntax for a while, you may come to the conclusion that Rust is too difficult of a language for you to figure out. There is a better way.

Rust has a really powerful feature called [enums](http://doc.rust-lang.org/book/compound-data-types.html#enums). Here is that same code using an enum:

```rust
enum ANSIColor {
    black,
    red,
    green,
    yellow,
    blue,
    magenta,
    cyan,
    white,
    default
}

fn color(fg: ANSIColor, bg: ANSIColor) -> String {
    use ANSIColor::*;

    let fg_code = match fg {
        black => 30,
        red => 31,
        green => 32,
        yellow => 33,
        blue => 34,
        magenta => 35,
        cyan => 36,
        white => 37,
        default => 39,
    };

    let bg_code = match bg {
        black => 40,
        red => 41,
        green => 42,
        yellow => 44,
        blue => 44,
        magenta => 45,
        cyan => 46,
        white => 47,
        default => 49,
    };

    let seq = format!("{};{}m", fg_code, bg_code);
    escape(seq.as_ref())
}
```

You can call the color method via `color(ANSIColor::black, ANSIColor::white);`. There is a complete, working example on [playpen](http://is.gd/OIRlyR).

I think the use of enums in that code is really expressive. Even more so than ruby: `color(:black, :white)`. The Rust enum provides the context of what `black` or `white` mean to the `color` function.  This also has the benefit of being type-checked by the compiler. If you or anyone else tries to specify a color like `ANSIColor::pink` the compiler would generate an error for you. This removes the pain of checking for valid colors at runtime, handling those errors at runtime and writing tests around those use-cases. This compile-time checking is what makes Rust such a powerful language.

There are plenty of cases for using dicts/hashes/maps in Rust. However, if you find yourself writing a function that accepts a finite set of options, then I suggest trying to use a Rust enum before resorting to a HashMap.
