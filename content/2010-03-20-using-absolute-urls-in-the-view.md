+++
title = "Using Absolute URL's In The View"
path = "/2010/03/20/using-absolute-urls-in-the-view.html"

[taxonomies]
tags=["closure", "factorymethod", "php53", "zendframework"]
+++

We recently had a project at work that involved replacing all the relative URL's from the application with absolute URL's.  In the past, developers had just hard-coded an absolute URL only when they need to force the browser over to https.  Now we are using multiple subdomains, so this approach is no longer sufficient.  We also wanted a way to easy rotate assets through multiple CDN URL's to speed up the time it takes a user's web browser to load all the content.
<!-- more -->
There are two requirements:
<ol>
	<li>Prefix a relative url path with a host.</li>
	<li>Rotate a set of relative url's through a given number of cdn hosts.</li>
</ol>
We currently use Zend Framework at work, so it is only logical we use as much of the framework as possible.&nbsp; The straight forward approach is to extend Zend_View_Helper_Abstract and create a few functions to fill the requirements.&nbsp; There are few problems with this approach.&nbsp; The first is function creep.&nbsp; We know we need to prefix http://www.hautelook.com and https://www.hautelook.com, but there may be more later.&nbsp; We may decide to use something like https://ssl.hautelook.com or require another subdomain like http://ftp.hautelook.com.&nbsp; This would require us to create another function, write some unit tests for it and finally send it through QA.&nbsp; This issue is even more compounded with the cdn rotation function.

Another problem is that we need absolute url's in some of our business logic.&nbsp; We have webservices that serve up XML or JSON that contain locations to such as images or a catalog.&nbsp; We want these services to take advantage of the absolute url logic too.&nbsp; If we implement a view helper, then the service layer becomes coupled to the view in order to reuse the logic.

Enter PHP 5.3, <a class="zem_slink" href="http://en.wikipedia.org/wiki/Functional_programming" title="Functional programming" rel="wikipedia">functional programming</a> and an inspirational post from <a href="http://eliw.wordpress.com/2010/03/10/an-intriguing-use-of-lambda-functions/">Eli White</a>.&nbsp; We can use the <a class="zem_slink" href="http://en.wikipedia.org/wiki/Factory_method_pattern" title="Factory method pattern" rel="wikipedia">factory method</a> pattern and closures to meet all the requirements and isolate the parts that change.&nbsp; Read <a href="http://www.hermanradtke.com/blog/php-goes-functional-in-version-5-3/">these</a> <a href="http://www.hermanradtke.com/blog/php-5-3-is-the-new-javascript-almost/">posts</a> if you are not familiar with functional programming in PHP.

<pre lang="php">
class Crimson_Url
{
    public static function absolute($host)
    {
        return function($path) use ($host) {
            return "{$host}{$path}";
        };
    }

    public static function rotate($subdomain, $host, $rotations, $protocol='http')
    {
        $current = 0;

        return function($path) use (&$current, $subdomain, $host, $rotations, $protocol) {
            if ($current == $rotations) {
                $current = 1;
            } else {
                $current++;
            }

            return "{$protocol}://{$subdomain}{$current}.{$host}{$path}";
        };
    }
}
</pre>

The absolute function demonstrates how simple, yet powerful, closures can be.&nbsp; The absolute function is creating a function that concatenates to strings together: a host name and a relative url.&nbsp; Here is how we would use this:

<pre lang="php">$www = Crimson_Url::absolute('http://www.example.com');
$ssl = Crimson_Url::absolute('https://www.example.com');

echo $www('/foo.jpg'), PHP_EOL;
echo $ssl('/secure.php'), PHP_EOL;
</pre>
Output:
<pre>http://www.example.com/items.php
https://www.example.com/secure.php
</pre>
Take this one step further and think about how you can use the HTTP_HOST or HTTP_REFERER values from $_SERVER to make absolute URL generation almost completely automatic.

The rotate function is slightly more complex.&nbsp; The $current variable is being declared in the factory method and passed by reference.&nbsp; We must explicitly pass this by reference otherwise PHP will pass it by value and the value of $current will be 1 everytime.&nbsp; Here is how we would use the rotater:

<pre lang="php">$cdn = Crimson_Url::rotate('cdn', 'hautelook.com', 3);
$sslCdn = Crimson_Url::rotate('cdn', 'hautelook.com', 3, 'https');

echo $cdn('/foo.jpg'), PHP_EOL;
echo $cdn('/bar.jpg'), PHP_EOL;
echo $cdn('/baz.jpg'), PHP_EOL;
echo $cdn('/bob.jpg'), PHP_EOL;

echo $sslCdn('/foo.jpg'), PHP_EOL;
echo $sslCdn('/bar.jpg'), PHP_EOL;
</pre>
Output:
<pre>http://cdn1.hautelook.com/foo.jpg
http://cdn2.hautelook.com/bar.jpg
http://cdn3.hautelook.com/baz.jpg
http://cdn1.hautelook.com/bob.jpg
https://cdn1.hautelook.com/foo.jpg
https://cdn2.hautelook.com/bar.jpg
</pre>
Notice how there are no problems with static variable conflicts.  Each function is independent and can be used for completely different tasks.

The complete source code, with tests and documentation, can be found here: <a href="http://github.com/hradtke/crimson/tree/master/Crimson_Url">http://github.com/hradtke/crimson/tree/master/Crimson_Url</a></pre>

<div style="margin-top: 10px; height: 15px;" class="zemanta-pixie"><a class="zemanta-pixie-a" href="http://reblog.zemanta.com/zemified/b602316b-6c98-4996-aba2-9daac249577a/" title="Reblog this post [with Zemanta]"><img style="border: medium none; float: right;" class="zemanta-pixie-img" src="http://img.zemanta.com/reblog_e.png?x-id=b602316b-6c98-4996-aba2-9daac249577a" alt="Reblog this post [with Zemanta]"></a><span class="zem-script more-related pretty-attribution"><script type="text/javascript" src="http://static.zemanta.com/readside/loader.js" defer="defer"></script></span></div>
