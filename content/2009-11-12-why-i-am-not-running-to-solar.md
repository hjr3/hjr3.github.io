+++
title = "Why I am not running to Solar"
path = "/2009/11/12/why-i-am-not-running-to-solar.html"

[taxonomies]
tags=["php", "solar", "zendframework"]
+++

Some of my thoughts on <a href="http://paul-m-jones.com/?p=1113">Paul M. Jones post about Solar and Zend Framework</a>.  This is less of a defense of Zend Framework and more of a commentary on Paul's framework ideas.

I favor design by contract so I can properly type-hint method parameters.  A framework should be written in a way I can safely extend the crap out of.  This is the exact use case of interfaces.

Universal constructors make the code harder to read.  I see this as a sign that inheritance is being used way too much.  Compose people, compose!

The registry pattern is not any better than a singleton.  They both have global scope.

I do like that there are so many ways to inject dependencies.  The MVC framework is also pretty nice.
