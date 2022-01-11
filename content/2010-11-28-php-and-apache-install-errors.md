+++
title = "PHP and apache install errors"
path = "/2010/11/28/php-and-apache-install-errors.html"

[taxonomies]
tags=["apache", "php"]
+++

I compile a few different versions of PHP on my development server.   Every once in a while I run into a problem with PHP installing correctly with apache.

The error looks something like this:
<blockquote>Installing PHP SAPI module: apache2handler
/usr/local/apache2/build/instdso.sh SH_LIBTOOL='/usr/local/apache2/build/libtool' libphp5.la /usr/local/apache2/modules
/usr/local/apache2/build/libtool --mode=install cp libphp5.la /usr/local/apache2/modules/
cp .libs/libphp5.lai /usr/local/apache2/modules/libphp5.la
cp .libs/libphp5.a /usr/local/apache2/modules/libphp5.a
ranlib /usr/local/apache2/modules/libphp5.a
chmod 644 /usr/local/apache2/modules/libphp5.a
libtool: install: warning: remember to run `libtool --finish /home/flumpy/temp/php-5.0.2/libs'
Warning! dlname not found in /usr/local/apache2/modules/libphp5.la.
Assuming installing a .so rather than a libtool archive.
chmod 755 /usr/local/apache2/modules/libphp5.so
chmod: cannot access `/usr/local/apache2/modules/libphp5.so': No such file or directory
apxs:Error: Command failed with rc=65536
.
make: *** [install-sapi] Error 1</blockquote>
I immediately start googling for a solution to this problem and get caught up in the symptoms.  The real problem is that there is a typo in the configure line.  In my most recent case, I had a '-' instead of a '--'.
