+++
title = "Please Do Not Interface the PHP World"
path = "/2011/10/22/please-do-not-interface-the-php-world.html"

[taxonomies]
tags=["interfaces", "java", "pdo", "php"]
+++

This started out as a quick response to a blog post titled <a href="http://pooteeweet.org/blog/2008">Interfacing the PHP World</a> by Lukas Kahwe Smith, but it quickly turned into a blog post of its own.

I think common interfaces would be bad for PHP. I find it ironic that a <a href="http://pooteeweet.org/blog/2008/2009#m2009">comment made by Stephen</a> is advocating a Java-like dictatorship to move PHP forward. PHP is much more successful and widely used than Java. It is Java should be learning from PHP, not the other way around. PHP is an ugly language with a mish-mash of syntax and conventions that makes the code not very pretty (especially compared to Python). PHP's only saving grace is that it actually gets things done. And luckily, for PHP, that is all that really matters.

Caching, logging and http clients may seem like simple things to standardize, but the mere fact that there are so many differing implementations is proof that they are not so simple. This reminds me of the SimpleCloud framework that tries to standardize the different cloud APIs: there is nothing simple about it! Furthermore, trying to standardize things can kill innovation and creativity. I fear the community will reject a new way to do things on the basis that the code does not implement the standard PHP interfaces. The Lithium framework, which takes a very different approach to frameworks, might not even exist if these interfaces were around a few years ago.

The most progress comes about when people reject or ignore the so called "truths" that surround them. The recent PHP fork is evidence of this. Forking PHP was a huge "no no" (despite the ironic fact that PHP is open source). That fork sparked both change and debate and had a positive effect. The author of the fork had tried to play by the rules setup by the PHP internals team and got nowhere. It was only when he decided to publish a fork that people (particularly those in internals) actually started paying attention.

This notion of interoperability between frameworks is just like trying to achieve nirvana. It is good to try, but never truly attainable. Look at PDO, a (supposedly) "lightweight, consistent interface for accessing databases in PHP". Except that it doesn't work with any of the NoSQL databases. One may argue that PDO is for SQL databases only, except that no one even considered that fact when it was being written. Simply read http://us2.php.net/manual/en/intro.pdo.php and it makes no mention of being SQL only. It assumed that SQL was all there was at the time of writing. The SQL-99 standard was the gospel for databases. This is the danger in defining standard interfaces. The advent of NoSQL made PDO largely irrelevant. The same fate will happen to any standard interface that we come up with today.

I encourage freedom the create new things. I don't want all of my frameworks playing by the same rules. If they are all using common interfaces, then we might as well merge all of the frameworks because the competition is over. Instead, let the adoption rate of the framework be the measure of success. I believe this is why PHP has been so successful. Rasmus has the courage to not try and define or predict what PHP needs. He lets the people using PHP decide what they need. This is in stark contrast to other people in internals that try to exert control of the process and define rigid rules and policies. PHP is not a purely procedural, object oriented or functional language. It is a mashup of all kinds of concepts and ideas. The cool thing is that it works.
