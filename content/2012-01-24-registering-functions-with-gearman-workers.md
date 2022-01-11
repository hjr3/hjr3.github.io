+++
title = "Registering Functions With Gearman Workers"
path = "/2012/01/24/registering-functions-with-gearman-workers.html"

[taxonomies]
tags=["gearman", "lamba", "php"]
+++

The Gearman <a href="php.net/manual/en/gearman.examples.php" target="_blank">examples on php.net</a> are a great primer for groking how the Gearman client and worker interact with each other. One gripe I have is that the examples declare global functions for the worker to register. I feel this leads develpers down the wrong path. With PHP5.3, there is an easier solution though: anonymous functions.<!-- more -->

Declaring a global functions in a gearman worker script may not seem like a big deal, but these things have a way of catching up to you. I personally ran into this when I suggested that HauteLook start using GearmanManager to manage the gearman workers. A side affect of this is that all gearman worker scripts now run in the same instance of PHP. There were a number of occasions when workers would fail to load on production because of the global naming conflicts.

I prefer to use register anonymous functions with my gearman workers. This keeps them out of the global scope and puts the logic right next to the worker registration call. This makes hard to test the logic inside the anonymous function, but I never test that logic. I treat the functions I register with gearman as controllers. I pass the workload off to a model class that is fully unit tested.

Here is a nice example:

<script src="https://gist.github.com/1673522.js?file=worker.php"></script>

I would update the Gearman examples on php.net, but I think a lot of people are still using PHP 5.2 (or even earlier). Providing multiple ways of registering the workers may confuse people too. Maybe next year.
