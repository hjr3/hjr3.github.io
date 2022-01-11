+++
title = "The Painful Gearman Upgrade Path"
path = "/2012/01/23/the-painful-gearman-upgrade-path.html"

[taxonomies]
tags=["gearman", "gearmand", "pecl", "php"]
+++

The Gearman project has been slowly migrating from C to C++. This migration has gone under the radar due to the popularity of Cent OS 5 and given gearmand version of 0.14. This version of gearmand worked with any version of pecl/gearman and there was never any compelling reason to upgrade gearmand. That changed with the release of pecl/gearman 1.0

<!-- more -->The release of pecl/gearman 1.0 included a fix to the exception handling of gearman workers. An uncaught exception would go unnoticed by the gearmand server before pecl/gearman 1.0. The only way to work around this was to catch all exception and explicitly return a failure code to gearmand. I fixed this by making by detecting an exception with the pecl/gearman code and sending a special exception message to the gearmand server. Unfortunately, the API for sending the exception message was introduced in gearman 0.21.

I did not think much of the new gearmand requirement when fixing this bug. Since that time I have been introduced to the painful upgrade process of gearmand. I think this pain is mostly felt by those using Linux distributions with long term support. The first hurdle to get over is the fact that gearmand 0.21 requires the boost 1.41 libraries. A default Cent OS install only provides boost 1.39 libraries. There was a 3rd party that provided boost 1.41 libraries for Cent OS, but they used a non-standard installation directory. After some experimenting, I did find a fix: <a href="http://groups.google.com/group/gearman/msg/12bcc13691e132ae">http://groups.google.com/group/gearman/msg/12bcc13691e132ae</a>.

The good news is that upgrading to Cent OS 6 makes compiling gearman much easier with the inclusion of boost 1.41 libraries by default. The next problem comes when trying to submit a patch. Cent OS 5 comes with aclocal 1.9 and the gearman project requires aclocal 1.11. Again, upgrading to Cent OS 6 makes life easier as it comes with aclocal 1.11 by default. I learned the hard way that the gearman project also requires autoconf 2.64, but Cent OS 6 only provides autoconf 2.63. The 2.64 version of autoconf provides a macro named m4_ifnblank which is required to build the configure script used to compile gearman. You can see me answer my own question here: <a href="https://answers.launchpad.net/gearmand/+question/185592">https://answers.launchpad.net/gearmand/+question/185592</a> regarding the autoconf issue.

These build issues have made it tough for users to upgrade to the pecl/gearman 1.x line. I think I will be forced to maintain the pecl/gearman 0.8.x line for some time with bug fixes as users struggle to get gearmand 0.21 on their Cent OS 5 servers. I have definitely learned to do more research on migration paths before requiring a newer dependency version.
