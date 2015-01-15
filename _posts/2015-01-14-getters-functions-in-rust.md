---
layout: post
title: "Getter Functions In Rust"
tags:
- rustlang
status: publish
type: post
published: true
---

As soon as I started writing implementations for structs in Rust I started fighting with the compiler. Writing what seemed like a simple getter function caused me a lot of frustration. The `self` parameter can really throw me off in Rust. I reflexively treat it like `this` in C++, which has no concept of `&` or `&mut`. I do this because I think of `impl Person` as defining methods on a _class_ as I would do in C++. This can be really misleading.

Consider this Ruby code that we want to port to Rust: 

{% highlight ruby %}
class Person
   attr_reader :name

   def initialize(name)
      @name = name
   end
end
{% endhighlight %}

In Rust this would look something like:

{% highlight rust %}
struct Person {
   name: String
}

impl Person {
   fn new(name: String) -> Person {
      Person { name: name }
   }

   fn get_name(self) -> String {
      self.name
   }
}

#[test]
fn test_get_person() {
    let p = Person::new("Herman".to_string());
    assert!(p.get_name().as_slice() == "Herman");
}
{% endhighlight %}

I run `rustc --test person.rs`, everything compiles and things are looking good. Even the test passes. What happens if I want to use `p` again though? If I modify my test to call `.get_name()` again I receive a cryptic error:

{% highlight rust %}
#[test]
fn test_get_person() {
    let p = Person::new("Herman".to_string());
    assert!(p.get_name().as_slice() == "Herman");
    assert!(p.get_name().as_slice() == "Herman");
}
{% endhighlight %}

{% highlight bash %}
$ rustc person.rs

person.rs:21:13: 21:14 error: use of moved value: `p`
person.rs:21     assert!(p.get_name().as_slice() == "Herman");
                         ^
<std macros>:1:1: 5:46 note: in expansion of assert!
person.rs:21:5: 21:50 note: expansion site
person.rs:20:13: 20:14 note: `p` moved here because it has type `Person`, which is non-copyable
person.rs:20     assert!(p.get_name().as_slice() == "Herman");
                         ^
<std macros>:1:1: 5:46 note: in expansion of assert!
person.rs:20:5: 20:50 note: expansion site
error: aborting due to previous error
{% endhighlight %}

I read about [ownership in the Rust Book](http://doc.rust-lang.org/book/ownership.html) and recall some of what _moved_ means, but it is not clear where to go from here. What many people new to Rust do is resort to using `.clone()`, but even that will not satisfy the compiler. Thinking back to C++, using a reference makes sense! Let's changing the first parameter to `.get_name()` from `self` to `&self`:

{% highlight bash %}
$ rustc person.rs

person.rs:13:7: 13:11 error: cannot move out of borrowed content
person.rs:13       self.name
{% endhighlight %}

Whenever I see the word _borrowed_ I know the compiler is referring to someething that is being passed by reference. In this case `&self` is being passed by reference. The compiler is trying to tell me that it cannot move ownership of `name` from my borrowed `&self`. I do not want to give up ownership of `name` though. I simply want my test to have access to the value for a little while. So, the next step is to return a reference to a String, via `&String`, so ownership doesn't change. Compiling that shows me:

{% highlight bash %}
$ rustc person.rs

person.rs:13:7: 13:16 error: mismatched types: expected `&collections::string::String`, found `collections::string::String` (expected &-ptr, found struct collections::string::String)
person.rs:13       self.name
                   ^~~~~~~~~
error: aborting due to previous error
{% endhighlight %}

This sort of error is very familar in Rust. Turning `self.name` into a reference via `&self.name` makes everything compile and leaves us with:

{% highlight rust %}
struct Person {
   name: String
}

impl Person {
   fn new(name: String) -> Person {
      Person { name: name }
   }

   fn get_name(&self) -> &String {
      &self.name
   }
}

#[test]
fn test_get_person() {
    let p = Person::new("Herman".to_string());
    assert!(p.get_name().as_slice() == "Herman");
    assert!(p.get_name().as_slice() == "Herman");
}
{% endhighlight %}

## Comparison to C++

What made things really click for me is to think about [how references work in C++](https://en.wikipedia.org/wiki/Reference_%28C%2B%2B%29#Uses_of_references). I also see the [Rust Book](http://doc.rust-lang.org/book/method-syntax.html) now includes the language _We should default to using `&self`, as it's the most common._ within the context of Rust's methods.
