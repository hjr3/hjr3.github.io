+++
title = "Don't Overuse Use"
path = "/2010/04/19/dont-overuse-use.html"

[taxonomies]
tags=["autoloading", "maintenance", "namespaces", "php53"]
+++

Nate Abele just announced <a href="http://rad-dev.org/lithium/wiki/blog/Lithium-0-9-The-Lambdas-are-awesome-Edition">Lithium 0.9</a> on the rad-dev blog.  I think Lithium is a great looking framework and can't wait for it to get a 1.0 release and really start to take off.  However, looking through the examples I started to notice that the use of the "use" namespace keyword was an often used convention.  It reminded me of the require/require_once creep of the days of old.  I was discussing it with Nate via Twitter, but I couldn't seem to get my point across.  Maybe I will have better luck here...

<!-- more -->Before PHP  went OO and autoloading existed, users had little choice but to place require_once statements throughout their code to satisfy the application dependencies.  At the start of a project this was fine as the dependencies were thoughtfully mapped out and the require_once statements strategically placed.  However, as time goes on and code goes through maintenance the require_once creep starts to set in.

What do I mean by require_once creep?  I simply mean that there are require_once statements including code that doesn't need to be included.  This caused a couple of problems:
<ol>
	<li>More code for the PHP parser to parse and thus a slower application</li>
	<li>Confusing code dependencies</li>
</ol>
Now, admittedly problem 2 is not the major issue problem 1 is.  However, nothing is more frustrating than removing a innocuous require_once from a file only to have some other random file break because it was including that file and depending on that require_once.  There is an entire PHP extension dedicated to helping people solve this very problem: <a href="http://us.php.net/inclued">inclued</a>.  I still remember Mozilla's conference talk about refactoring TikiWiki to not simply include every nearly every single library file on every single page.

To fix this problem, autoloading was introduced.  This allowed people to stop using require_once and spawned a method of pseudo-namespacing files.  One of the more popular methods of psuedo-namespacing was the PEAR standard.  This standard said to name the class based on the directory location of the class.  So if I have class in herman/awesome/Solution.php I would name the class Herman_Awesome_Solution.  This obviously gave rise to some pretty long class names, but people generally loved it as it saved them from require_once creep.

Let's fast forward to PHP 5.3.  The "use" keyword is introduced to let users alias long namespaces to a shorter name.  So I can reference the namespace \this\is\a\really\long\namespace as longns.  This is great, now I can reference a really long namespace with a single word.  All I have to do is remember to alias it.  And the aliases are descending, so if I alias a class that alias two other classes, I can use those additional aliases free of charge.

But wait, doesn't it seem like we just went full circle with naming conventions?  We started with simple class names with require_once creep, graduated to autoloading and using super long class names and are now back using simple class names with use creep.  The use creep doesn't have the performance halting side effects of require_once, but it sure seems like it can cause quite a dependency maintenance nightmare.  It may be no problem for you, but what about the guy coding next to you who really doesn't care?

I think I will stick to using the fully qualified namespace (FQN) of a class.  It will take a few more keystrokes, but it keep my dependencies cleaner.
