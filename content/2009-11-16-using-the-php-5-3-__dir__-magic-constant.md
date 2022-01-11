+++
title = "Using the PHP 5.3 __DIR__ magic constant"
path = "/2009/11/16/using-the-php-5-3-__dir__-magic-constant.html"

[taxonomies]
tags=["php", "php53"]
+++

There is a new magic constant in PHP 5.3: __DIR__.  This new constant does not actually do anything new.  It replaces the use of dirname(__FILE__) that is commonly found  code using prior PHP versions.

<!-- more -->

One of the troubles with including files is that the current directory is constantly a moving target.  Consider the following structure:

<pre>
index.php
admin/index.php
includes/
    foo.php
    bar.php
</pre>

Imagine for a moment that both index.php and admin/index.php include includes/foo.php.  If foo.php includes bar.php, the relative directory is either "../" or "../admin/" depending on whether index.php or admin/index.php was executed.  The use of the dirname() function along with the __FILE__ magic constant solved this problem for a long time.

The __FILE__ magic constant gives the full path and filename of the current file.  The dirname() function gives the directory name of a file path.  This was used prior to PHP 5.3 to solve the include problem described above.

<pre lang="php">
<?php
// foo.php
require_once dirname(__FILE__) . '/bar.php';
?>
</pre>

In PHP 5.3, this has been super-ceded by the use of the __DIR__ magic constant.

<pre lang="php">
<?php
// foo.php
require_once __DIR__ . '/bar.php';
?>
</pre>

Why the change?  Well, it is more succinct and more clear on what is going on.  Also, the magic constant is part of the Zend engine, so it is a little more performant as well.  
