+++
title = "Working with the PHP source tree"
path = "/2011/05/02/working-with-the-php-source-tree.html"

[taxonomies]
tags=["pecl", "php", "svn"]
+++

The svn repository for PHP is rather large.  Trying to checkout the entire repo is both time consuming and wastes a lot of space.  Most people, including myself, are only concerned with a subset of the repository.  One of the advantages svn has over git is the ability to do partial checkouts of the repository.  I am going to borrow from an old <a title="svn checkout suggestion" href="http://www.mail-archive.com/internals@lists.php.net/msg43154.html" target="_blank">email</a> Rasmus sent that details how to do a partial checkout of the PHP source code.<!-- more -->I mostly work on pecl/memcache and pecl/gearman and send the occasional patch for a php core extension.  I will ignore any other branches of the PHP svn repository that I am not currently working on.

Here is my checkout process:
<ul>
	<li><strong>svn co http://svn.php.net/repository --depth empty src</strong>
<ul>
	<li>Checkout the intial, empty, repository</li>
</ul>
</li>
	<li><strong>svn co http://svn.php.net/repository/php http://svn.php.net/repository/pecl --depth immediates src</strong>
<ul>
	<li>Checkout the immediate files and directories in the php and pecl directories of the repository.  This will checkout empty directories for all the pecl extensions and empty directories for the php modules.</li>
</ul>
</li>
	<li><strong>cd src/php/php-src</strong></li>
	<li><strong>svn up branches tags --set-depth immediates </strong>
<ul>
	<li>Checkout empty directories for all branches and tags under php-src.  Any branches or tags created in the future will automatically be created as an empty directory on a future svn update.</li>
</ul>
</li>
	<li><strong>svn up trunk branches/PHP_5_3/ --set-depth infinity</strong>
<ul>
	<li>Checkout all the source code for trunk and the PHP 5.3 development branch.</li>
</ul>
</li>
	<li><strong>cd ../../pecl/</strong></li>
	<li><strong>svn up gearman memcache --set-depth infinity</strong>
<ul>
	<li>Checkout trunk, branches and tags for the pecl/gearman and pecl/memcache extensions</li>
</ul>
</li>
</ul>
The great thing about this approach is that I can show and hide other  parts of the PHP svn repository at will with only a few svn update  commands.  I regularly add an remove other parts of the PHP svn repository at will.  Combined with <a href="http://lxr.php.net/">http://lxr.php.net/</a>, I can quickly find examples and code I am looking for.
