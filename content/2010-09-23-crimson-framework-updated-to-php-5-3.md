+++
title = "Crimson Framework Updated to PHP 5.3"
path = "/2010/09/23/crimson-framework-updated-to-php-5-3.html"

[taxonomies]
tags=["crimson", "framework", "php5.3"]
+++

I have updated my <a href="http://github.com/hradtke/crimson">Crimson</a> framework to use PHP 5.3 namespace support instead of the old PEAR style class namespacing.  I did this primarily as an exercise in migrating a code-base to use namespaces.  Just in case anyone was relying on the old framework code, there is a pre-5.3 branch on github.<!-- more -->

Along the way I decided to deviate from the separate component structure and merge all components into a single framework structure.  It is still my goal to keep dependencies within the framework to a minimum, but I felt the extra work required to maintain each component separately was not worth it.  This was especially the case for the unit tests.  Each component required a considerable amount of boilerplate code to bootstrap and run the tests.  I was able to reduce the amount of boilerplate considerably by merging the tests into one suite.  I have been following the developments of Zend Framework 2.0 to see what major changes they are making.  I think the large overhaul to the ZF2 tests section was a significant influence on my decision to do some overhaul myself.

I have also added some additional features to the framework that I will be talking about in a later post.
