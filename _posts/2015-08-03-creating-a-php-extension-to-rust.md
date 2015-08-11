---
layout: post
title: "Creating a PHP Extension to Rust"
tags:
- rustlang
status: publish
type: post
published: true
---

I am going to walk through the creation of a PHP extension that works with a Rust library. I have a [working example][working example] here. I also created a PHP extension for my [Rust selecta port][Rust selecta port]. Both examples use the same foreign function interface (ffi). I made sure to pick an example that uses strings because strings add additional complexity that numbers do not introduce.

## Before Getting Started

note: I created a [docker container][docker container] that will set environment up.

You are going to need a development version of PHP. You can test if you have it by running:

```
$ which phpize
```

If the `which` command finds something like `/usr/local/bin/phpize`, you are in business. If you do not have it, I believe you can run `yum install php-devel` on CentOS or `apt-get install php5-dev` on Debian/Ubuntu. You can also compile PHP from source to get it.

## Compiling The Extension

Our [Rust library][Rust library] exposes a single function named `ext_score`. It takes two parameters of `*const char` and returns a 64-bit floating point type (or a double). To build the Rust library:

```
$ cd rust
$ cargo build
```

Our [PHP extension][PHP extension] defines a single function named `score` that will glue PHP userland to our `ext_score` Rust function. To build the PHP extension:

```
$ cd php-ext
$ phpize
$ ./configure --with-score=../rust/target/debug
$ make
$ php -d extension=modules/score.so -r "var_dump(score('vim', 'vi'));"
```

Now that we have a working example, we can explore what each of the files are actually doing.

## Configuring The Extension

I am going to dive right into the autotools stuff. I think autoconf is magic and the PHP wrappers around autoconf is _dark magic_. However, it is the biggest hurdle to getting a PHP extension working. All the stuff going on here is dense and it would take a whole blog post to go through it enough detail. You can usually get away with copy/pasting this stuff and tinkering with it so it works. I will try and touch on a number of things that have tripped me up though. If you make it through this section, the rest is easy.

If this is way more than you need, feel free to just start hardcoding stuff in your extension. [I did][I did]! You can then skip down to where I start talking about the source code.

Here is the `config.m4` file I wrote for my extension. Let us walk through what is going on inside here.

```
PHP_ARG_WITH(score,
    [whether to enable the "score" extension],
    [  --enable-score          Enable "score" extension support])

if test "$PHP_SCORE" != "no"; then

    SEARCH_PATH="/usr/local /usr"
    SEARCH_FOR="libscore.so"
    if test -r $PHP_SCORE/$SEARCH_FOR; then # path given as parameter
      SCORE_LIB_DIR=$PHP_SCORE
    else # search default path list
      AC_MSG_CHECKING([for score files in default path])
      for i in $SEARCH_PATH ; do
        if test -r $i/lib/$SEARCH_FOR; then
          SCORE_LIB_DIR=$i
          AC_MSG_RESULT(found in $i)
        fi
      done
    fi

    if test -z "$SCORE_LIB_DIR"; then
      AC_MSG_RESULT([not found])
      AC_MSG_ERROR([Please reinstall the score rust library])
    fi

    PHP_CHECK_LIBRARY(score, ext_score,
    [
        PHP_ADD_LIBRARY_WITH_PATH(score, $SCORE_LIB_DIR, SCORE_SHARED_LIBADD)
        AC_DEFINE(HAVE_SCORE, 1, [whether ext_score function exists])
    ],[
        AC_MSG_ERROR([ext_score function not found in libscore])
    ],[
        -L$SCORE_LIB_DIR -R$SCORE_LIB_DIR
    ])

    PHP_SUBST(SCORE_SHARED_LIBADD)
    PHP_NEW_EXTENSION(score, score.c, $ext_shared)
fi
```

The `config.m4` file is a mix of bash, some autoconf (AC) functions and some custom PHP functions. At a high level, we are writing some code to detect where our Rust library exists and then add that information into a `Makefile` that we will auto-generated. That `Makefile` is generated from a script called `configure`. The majority of the `configure` script is going to be created for us by PHP tooling. However, we need to add some extension specific information.

Let us start by hooking our extension into the `configure` script using `PHP_ARG_WITH`. The `PHP_ARG_WITH` function takes three parameters:

   1. The name of the extension. This will be used to determine the name of our extension variable. In this case, `$PHP_SCORE`.
   1. The human readable string shown when `./configure --with-score` is run. Example: _checking whether to enable the "score" extension... yes, shared_
   1. The human readable string shown when `./configure --help` is run. This is why the spacing of the string is a bit odd.

Now we can run `./configure --with-score` and the `configure` script will know what we are talking about. Next, we need to tell the configure script where to find our library so it can add those details to the Makefile.

### No Header File

PHP assumes that a library comes with a header file that describes the functions a library exposes. Rust's FFI does not provide a header file. If we were working with a library, such as [gearman][gearman], then we would expect `/usr/include/gearman.h` to exist. The standard PHP `config.m4` file uses this header file to check if a library is installed or not. To work around this lack of a header file, we can look for the shared object file instead: `SEARCH_FOR="/lib/libscore.so"`. Now that we have a Rust compatible file to check for, we need to start searching for it.

Before we start checking for our `libscore.so` shared object in commonly used directories like `/usr` and `/usr/local`, we want to first allow an override via `./configure --with-score=/path/to/library`. This is really useful when working on our Rust library in conjunction with the PHP extension. I can run `cargo build` and that will install `libscore.so` in `/home/herman/projects/selecta/php-ext/target/debug/`. I can then configure my PHP extension using `./configure --with-score=/home/herman/projects/selecta/php-ext/target/debug/`. When I specify a path like this, the path will be stored in the `$PHP_SCORE` variable. This saves us from having to repeatedly _install_ our Rust library. If no override was specified, we can start searching some common places. Feel free to add more directories to search for, such as `/opt/local`.

### Validating Before Linking

We have located a file called `libscore.so`, but we need to make sure it is a valid library file. The `PHP_CHECK_LIBRARY` function is used to validate our shared object contains a known function, or _symbol_. The `PHP_CHECK_LIBRARY` function takes five parameters:

   1. The name of the library. In our case _score_ will be transformed into `-lscore` when compiling. Example: `cc -o conftest -g  -O0  -Wl,-rpath,/usr/local/lib -L/usr/local/lib -lscore conftest.c`
   1. The name of the function to try and find within our _score_ library.
   1. The set of actions to take if the function is found. In our case, we are adding to the Makefile code to compile against our library and defining `HAVE_SCORE` which is used by the during compilation.
   1. The set of actions to take if the function is not found. In our case, we are throwing an error with a human readable error message.
   1. The set of extra library definitions. In our case, we are making sure the compiler knows where to find our shared object.

The `PHP_ADD_LIBRARY_WTH_PATH` function takes three parameters:

   1. The name of the library.
   1. The path to the library.
   1. The name of a variable to store library information. We will use this with `PHP_SUBST`.

### Final Steps

We are almost there!

The `PHP_SUBST` function adds a variable with its value into the Makefile.

The `PHP_NEW_EXTENSION` function takes a lot of parameters, but I am only going to go over three:

   1. The name of the extension
   1. The list of sources, or files, used to build the extension.
   1. Whether the extension should be dynamically loaded or statically compiled. The `$ext_shared` variables sets this to the proper value.

### Building Your Own Extension

Normally, you can use the [ext_skel][ext_skel] program to create an PHP extension out of the box. However, the `ext_skel` generated `config.m4` file makes some assumptions that Rust violates. It is a good starting point though. Change to the directory where you want the extension to be created and then run `ext_skel`:

```
$ cd /path/to/projects
$ /path/to/php-src/ext/ext_skel --ext-name=php-rust-ext
```

This will create a `/home/herman/projects/php-rust-ext` directory with the following files: `config.m4  config.w32  tests`. I did not go over `config.w32` as that is for Windows and I am woefully ignorant when it comes to PHP and Windows. The `config.m4` has a lot of the comments to help you and you have my notes above to make any necessary changes.

### Making Changes To config.m4

Once you think you have the `config.m4` file properly setup, run the `phpize` command. This program will add a bunch of auto-generated files to your directory. Feel free to `.gitignore` them and do not check them into version control. Most importantly, it creates our `configure` file which we will now use to generate our `Makefile`.

You will need to make changes to the `config.m4` to get your specific extension working with your library. If you make a change to the `config.m4` file, then make sure you run `phpize` again. If you make a change and then just run `./configure --with-score` then you will not get the benfit of your changes.

## Extension Header File

Here is the standard PHP header file for an extension. The convention is to use `php_[extension-name].h` as the name. In our case, `php_score.h`.

```c
#ifndef PHP_SCORE_H

#define PHP_SCORE_H

#define PHP_SCORE_EXTNAME "score"
#define PHP_SCORE_EXTVER  "1.0"

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "php.h"

extern zend_module_entry score_module_entry;
#define phpext_score_ptr &score_module_entry

// Define our Rust foreign function interface (ffi) here
extern double ext_score(unsigned char *, unsigned int, char *, unsigned int);

#endif
```

You can copy/paste most of this and replace `SCORE` and `score` with the name of your extension. I chose to define the score libraries functions here. We are telling the compiler that something external to our code is defining a function named `ext_score`. This allows our code to compile successfully when we go to use this Rust function. Make sure you list all the functions your Rust library is exposing.

## Extension Source Code

The `score.c` file is a little long and most of it is uninteresting. The full [score.c][score.c] file is here. Let us explore the relevant portion where we create a PHP userland function named `score` to call our Rust `ext_score` function.

```
PHP_FUNCTION(score)
{
    char *choice;
    int choice_len;
    char *query;
    int query_len;

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "ss", &choice, &choice_len, &query, &query_len) == FAILURE) {
        return;
    }

    double s = ext_score(choice, choice_len, query, query_len);

    RETURN_DOUBLE(s);
}
```

We declare new PHP functions using the `PHP_FUNCTION` macro and pass it the name of the function. If you are using gdb and you want to break on this function, the macro transforms it into `zif_[func-name]`. In our case: `zif_score`. The `zif` stands for _Zend Interface Fucntion_. You will notice the word _Zend_ being used a lot as that is the name of the PHP virutal machine (and the name of the company whose founders built the vm).

We are using the `zend_parse_parameters` function to parse the paramters being specified in our userland function. In this case, we are expecting two strings. If this function looks a little gnarly, well that is because it is. I will provide some links at the end that explain how this function works in more detail. Suffice to say, we get back two non-nul terminated `char *` values and their corresponding lengths as `int`s.

We can pass the strings into our `ext_score` function, get a result back and then return that value to userland PHP. We now have a working end-to-end PHP extension to a Rust library.

## Further Reading

For some detail on the PHP (or Zend) specific functions and macros:

   * [Extension Writing Part I: Introduction to PHP and Zend][devzone 1]
   * [Extension Writing Part II: Parameters, Arrays, and ZVALs][devzone 2]

If you are really serious about building PHP extensions, I suggest purchasing Sara Goleman's excellent book on [Extending and Embedding PHP].

[working example]: https://github.com/hjr3/rust-php-ext
[Rust selecta port]: https://github.com/hjr3/selecta/tree/php-ext
[docker container]: https://hub.docker.com/r/hjr3/rust-php-ext/
[Rust library]: https://github.com/hjr3/rust-php-ext/blob/master/rust/src/lib.rs
[PHP extension]: https://github.com/hjr3/rust-php-ext/blob/master/php-ext/score.c
[ext_skel]: https://github.com/php/php-src/blob/master/ext/ext_skel
[I did]: https://github.com/hjr3/selecta/commit/b48de0ae95618447a5d237bf48e2dbd8ac45e203#diff-ec28c2fa28e17d40dbe2bee40768b51fR7
[score.c]: https://github.com/hjr3/rust-php-ext/blob/master/php-ext/score.c
[devzone 1]: http://devzone.zend.com/303/extension-writing-part-i-introduction-to-php-and-zend/
[devzone 2]: http://devzone.zend.com/317/extension-writing-part-ii-parameters-arrays-and-zvals/
[Extending and Embedding PHP]: http://www.amazon.com/Extending-Embedding-PHP-Sara-Golemon/dp/067232704X
[gearman]: http://gearman.org/
