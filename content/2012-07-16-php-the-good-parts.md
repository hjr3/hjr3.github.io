+++
title = "PHP: The Good Parts"
path = "/2012/07/16/php-the-good-parts.html"

[taxonomies]
tags=["php"]
+++

This blog post is inspired by Douglas Crockford's book <a href="http://www.amazon.com/JavaScript-Good-Parts-Douglas-Crockford/dp/0596517742">JavaScript: The Good Parts</a>.

All programming languages have warts. That is, there were certain decisions made about a language that are less than ideal. Some people are driven to remove these warts from the language in an attempt to make the language better. I think this is done with the best intentions, but can often have negative consequences. JavaScript is a good example of a language that has a lot of warts. Despite all these warts, JavaScript is a very useful language and has seen a huge rise in popularity. I feel PHP is the same way. It has bad parts, but there are so many good parts that we need to celebrate those good parts. Here are a few things off the top of my head:
<h2>Arrays</h2>
I think the array is the single most powerful and useful part of PHP. The PHP array is the Swiss Amry knife in my programming toolkit. I have written applications in a number of other software languages and I have yet to find anything else more useful. The best part about arrays is that they just work. I don't have to decide ahead of time between a list or a map. The PHP array is to data structures as NoSQL is to SQL. Better still is that PHP core uses them all over the place. Results from the database: arrays. Parsing a json POST from the client: arrays. They are ubiquitous in PHP in both core and userland. I cannot say enough good things about PHP arrays.
<h2>Web Ready</h2>
PHP is web ready. I do not mean that PHP is easy to integrate into a webserver. PHP is easy to integrate, but I think a lot of languages do a good job of integrating to webservers now. I mean PHP is built for the web. It is so easy to create an HTML template and pass the data to it. I think Mustache and Twig are great. That being said, I do not have to decide on a templating language in order to get up and running. Everyone understands HTML.

I do think this is feature is getting less important as the web develops. I write a lot of API's and send almost everything to the client via json. However, they are still tons of websites out there are that are not platforms and need to serve up HTML.
<h2>Streams</h2>
Streams are the best kept secret in PHP. Most people do not even realize they are using streams when they are interacting with file systems or networks. I wrote a <a href="https://gist.github.com/1706840">plugin to push messages</a> to the Phergie IRC bot in less than an hour using streams. They are a really powerful abstraction that is used all over PHP.
<h2>Type Juggling</h2>
For the most part, a web application is just a bunch of strings. HTTP is all strings, most database adapters return strings and all output is strings. PHP handles all of this and removes all kinds of boilerplate code from my applications. I think PHP has the most sensible implementation of juggling too. Yes, they are some problems with large integers that are represented as strings. It is by no means perfect. However, I think the PHP zval has saved me orders of magnitude more hours than pain.

For all the warts, there is plenty of beauty in PHP. I still enjoy writing web applications using PHP and focus my time using the parts of PHP that work really well.
