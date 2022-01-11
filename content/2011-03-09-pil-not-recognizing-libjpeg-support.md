+++
title = "PIL not recognizing libjpeg support"
path = "/2011/03/09/pil-not-recognizing-libjpeg-support.html"

[taxonomies]
tags=["gcc", "image", "libjpeg", "pil", "python"]
+++

PIL is python's imaging library.  I use this library when doing any sort of image processing.  Recently I had to install PIL on a new Cent OS 5 server.  I think pythons module installation process is awesome and almost never have problems. However, this particluar version of PIL 1.1.7 was giving me some problems with libjpeg. The libjpeg rpm was installed, but PIL was not getting built with libjpeg support.  This was very confusing to me because the setup summary said it jpeg support was available and the selftest.py script passed all the tests.  <!--more-->

After fishing around for a while I noticed the following warning during the build process:

<code>/usr/bin/ld: skipping incompatible /usr/lib/libjpeg.so when searching for -ljpeg</code>

The gcc line responsible for that warning had the following flags for linking:
<code>-L/usr/lib -L/usr/lib64 -ljpeg</code>

I am working on a 64bit system and apparently the linker was trying to link a 32bit libjpeg shared object instead of the 64bit one.  I copied the gcc statement, removed "-L/usr/lib" from it and manually compiled the _imaging.so file.  I had to then manually install (copy) the _imaging.so file into the to /usr/lib64/python/site-packages/PIL after that.  This finally solved the problem.
