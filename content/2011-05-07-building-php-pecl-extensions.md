+++
title = "Building php pecl extensions"
path = "/2011/05/07/building-php-pecl-extensions.html"

[taxonomies]
tags=["gdb", "pecl", "php", "svn"]
+++

My <a title="Working with the PHP source tree" href="http://www.hermanradtke.com/blog/working-with-the-php-source-tree/" target="_blank">last post</a> I explained how to efficiently checkout the php svn repository.  Now we need to start building pecl extensions and even php itself.  I prefer to use Cent OS for my linux needs and naturally use rpm's to track all my packages.  This means I have a stable version of php installed with all the various extensions that I could want.  Rather than messing with this stable version, I am going to build a custom debug build of php in /usr/local.  I say "debug", because this build of php will use the --enable-debug option to allow easy debugging using gdb.  Since I am doing pecl extension development, I don't want to build the trunk version of php.  I want to build my pecl extensions against the most recent stable version of php to isolate environmental issues as much as possible.<!-- more -->

Now building php from an svn tag is more tricky than building from a snapshot.  The goal is to start building extensions, so rather than waste time navigating the hurdles of building from svn, I will just grab the latest stable version of php from php.net.  Building php becomes very similar to any other program you have compiled from source.

<code>
wget http://www.php.net/get/php-5.3.6.tar.gz/from/this/mirror
tar xzf php-5.3.6.tar.gz
cd php-5.3.6
./configure --prefix=/usr/local/php-5.3.6 --with-config-file-path=/usr/local/php-5.3.6/etc --enable-debug
make
make test #optional
sudo make install
sudo cp php.ini-development /usr/local/php-5.3.6/etc/php.ini
sudo ln -s /usr/local/php-5.3.6 /usr/local/php
</code>

You will notice that I specified a version specific prefix for php to be installed at.  I do this so I can build multiple versions without clobbering an older build.  However, this makes maintaining any helper scripts a pain as the version continually changes.  To make things easier, I created a symlink so I can specify /usr/local/php for any helper scripts.  You may also want to add /usr/local/php/bin to your PATH in .bash_profile.

Now let's compile the pecl/memcache extesnsion.
<code>
cd php/src/pecl/memcache/trunk
phpize
./configure --enable-debug
make
make test
</code>

The phpize script is a bash script that bootstraps the environment for the pecl extension based on your installed version of php.  If you have multiple php versions installed, it is very important that you are using the correct phpize script.  I suggest using 'which phpize' to make sure that /usr/local/php/bin/phpize is the script you are using.

Now you will notice that I did not perform a 'make install'.  I generally don't install my extensions while developing with them.  I sometimes bounce back and forth between a few different version of an extension trying to fix various bugs and implement new features.  I prefer to specify the full path to the extension in my php.ini file.  The extension line will look something like this:
<code>
extension=/home/hradtke/projects/php/src/pecl/memcache/trunk/modules/memcache.so
</code>
Once that line is added you can verify the extension is loaded properly by running 'php -m'.  If you want more detail, you can run 'php -i'.

Now you can start playing with the code.  If you are unsure where to start, I suggest reading these articles:
<ul>
	<li><a href="http://devzone.zend.com/node/view/id/1021" target="_blank">Extension Writing Part I: Introduction to PHP and Zend</a></li>
	<li><a href="http://devzone.zend.com/node/view/id/1022" target="_blank">Extension Writing Part II: Parameters, Arrays, and ZVALs</a></li>
	<li><a href="http://devzone.zend.com/node/view/id/1024" target="_blank">Extension Writing Part III: Resources</a></li>
	<li><a href="http://blog.golemon.com/2006/06/what-heck-is-tsrmlscc-anyway.html" target="_blank">What the heck is TSRMLS_CC, anyway?</a></li>
	<li><a href="http://www.phpbuilder.com/manual/en/zend.variables.php" target="_blank">Zend API: Hacking the Core of PHP</a></li>
	<li><a href="http://talks.somabo.de/200711_php_code_camp.pdf" target="_blank">Extending PHP</a></li>
</ul>
If you get through all that and still want to know more, I suggest purchasing Sara Goleman's book <a href="http://www.amazon.com/Extending-Embedding-PHP-Sara-Golemon/dp/067232704X" target="_blank">Extending and Embedding PHP</a>.
