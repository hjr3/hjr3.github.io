+++
title = "PECL memcache 2.2.6 and 3.0.5 released"
path = "/2010/10/03/pecl-memcache-2-2-6-and-3-0-5-released.html"

[taxonomies]
tags=["memcache", "pecl", "php"]
+++

I just released <a href="http://pecl.php.net/package/memcache">memcache</a> versions <a href="http://pecl.php.net/get/memcache-2.2.6.tgz">2.2.6</a> and <a href="http://pecl.php.net/get/memcache-3.0.5.tgz">3.0.5</a> in PECL.  The 3.0.5 release fixed the delete weight bug that prevented people from upgrading to the latest version of the memcached daemon.  I know this was a major issue for many shops and I hope it will allow people to continue to use the 3.0.x branch as we try to finish the non-blocking i/o changes.

These two releases are my first releases as part of the development team working on PECL memcache and the first releases in almost 20 months.  I took over, along with Pierre-Alain Joye, when Antony Dovgal and Mikael Johansson went inactive back in March 2010.  As luck would have it, I suddenly became busy myself.  Inheriting a project, especially one that is half-done, takes quite a bit of time to get comfortable with.  The 7 months since I have started have flown by and I wish I had gotten more done.  Now that some critical bug fixes are out of the way, I hope to focus more on the non-blocking i/o branch development.

Pierre and I are working on a roadmap that focuses on getting much of the 3.0.x code into a stable state.  The 2.2.x branch will probably not see any new development, but will continue to be maintained with bug fixes.
