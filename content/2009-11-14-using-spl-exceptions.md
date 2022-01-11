+++
title = "Using SPL Exceptions"
path = "/2009/11/14/using-spl-exceptions.html"

[taxonomies]
tags=["exceptions", "php", "spl"]
+++

Brandon Savage has a <a href="http://www.brandonsavage.net/exceptional-php-nesting-exceptions-in-php/">great</a> <a href="http://www.brandonsavage.net/exceptional-php-extending-the-base-exception-class/">series</a> of <a href="http://www.brandonsavage.net/exceptional-php-introduction-to-exceptions/">posts</a> on using exceptions in PHP.  Unfortunately, he does not introduce the SPL exceptions into the discussion.

<!-- more -->

The Standard PHP Library (SPL) has quite a few exception <a href="http://www.php.net/manual/en/spl.exceptions.php">classes</a> that are useful right out of the box.  Additionally, these exceptions can be extended just like the Exception and ErrorException classes.

My favorite is the <a href="http://www.php.net/manual/en/class.invalidargumentexception.php">InvalidArgumentException</a>.  I use this exception if some parameter to a function or method is not what I expected.

<pre lang="php">
<?php

function doubleMe($number)
{
    if (!is_numeric($number)) {
        throw new InvalidArgumentException(
            'Unable to double a non-numeric value');
    }
}
</pre>

If you are using the __call() magic method, the BadMethodCallException is another great built-in exception to use.

<pre lang="php">
class Foo
{
    public function __call($name, $params)
    {
        if (!method_exists($this, $name)) {
            throw new BadMethodCallException(
                "Method $name does not exist");
        }
    }
}
</pre>

The SPL exception classes are a great alternative to writing your own exceptions classes.  For some more examples you can check out PHPUnit.  Sebastian Bergmann uses a lot of SPL exceptions in his code there.
