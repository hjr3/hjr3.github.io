+++
title = "SPL FilterIterator in the real world"
path = "/2011/12/15/spl-filteriterator-in-the-real-world.html"

[taxonomies]
tags=["functional", "iterator", "oo", "php", "spl"]
+++

The Standard PHP Library (SPL) is a powerful set of tools that are often overlooked. It is very common to see an SPL talk at conferences, but those talks usually just introduce each SPL class to the audience without giving some real world examples. I am going to show you a real world example on how to use SPL FilterIterator in an ecommerce website.
<!-- more -->
This particular ecommerce website sells actual goods. One problem with selling actual goods, instead of virtual goods, is the supply can run out. I have a simple use-case where I don't want to display an item that is sold out. Consider the following example data:

&nbsp;
<script src="https://gist.github.com/1483101.js?file=data.php"></script>
&nbsp;

This is a pretty common use case and I am sure most people would write the logic something like this:

&nbsp;
<script src="https://gist.github.com/1483101.js?file=procedural.php"></script>
&nbsp;


There is nothing logically wrong with this code. It is a very readable and easy to understand way to write it. With the rise in popularity of JavaScript and other functional languages, some of us may take a different approach. Using the underscore framework (available for both PHP and JavaScript), you could also write it like this:

&nbsp;
<script type="text/javascript" src="https://gist.github.com/1483101.js?file=functional.php"></script>
&nbsp;

The use of a callback function should be familiar to anyone with at least a basic knowledge of JavaScript. The callback function is evaluating each item in the iterator to true or false. The major drawback to both of the code snippets is that the logic for determining whether or not an item is sold out is not able to be re-used. The use of procedural code in the first example and the use of an anonymous function in the second example also make it hard to test. We can improve the second example by not using an anonymous callback:

&nbsp;
<script type="text/javascript" src="https://gist.github.com/1483101.js?file=functional2.php"></script>
&nbsp;

Now I can re-use this function and easily test it. We can use SPL FilterIterator in a very similar way to the functional example:

&nbsp;
<script src="https://gist.github.com/1483101.js?file=spl.php"></script>
&nbsp;

Now my logic for what constitutes a soldout item is isolated in a class. This coincides with the object oriented principle of "encapsulate what varies". I think the biggest stumbling block for using SPL is that it just seems too heavy. You might be wondering why you should go to the trouble of creating a new class to perform such a simple task. The procedural example above is faster to write. Some of us might be tempted to use the functional example with an anonymous function because it feels more "expressive". Now consider what happens when the ecommerce system introduces returns. The new formula for determining a soldout item is now:

<code>
availability = (purchased - sold) + returned
</code>

This isn't some hypothetical example, it actually happened. It was real easy to update the logic to handle returns and make sure the tests passed.

The last decision you have to make is whether or not to put the logic in a database query. If there is a huge performance boost by writing the logic into the query, it may be worthwhile. A query is still testable, but you have to setup some test data in the database in order to test it (which means you probably won't do write the test for it). It is harder to re-use queries, so the business logic for determining sold out items may be duplicated over a number of queries too. You also might want to consider what happens if you decide to alleviate the database load by putting the list of items for sale in a cache (like APC or memcache).

The SPL classes, especially FilterIterator, really start to shine when dealing with a dataset outside of our control. More and more platforms are becoming service based and we have no control over how the data comes back. Especially when you consider something like the Twitter API timeline response and trying to filter out any tweet that starts with "RT".
