+++
title = "Scaling URL's"
path = "/2009/11/15/scaling-urls.html"

[taxonomies]
tags=["cdn", "php", "scaling", "url"]
+++

Using non-relative URL's during early development can alleviate a lot of growing pains.  This may seem counter-intuitive at first, but hear me out.  We all learned long ago to stop hard-coding the domain name into the href attribute of an anchor tag.  Instead, we used relative URL's such as '/index.php' to make our code much more portable.  However, relative URL's become a pain point when trying to scale your website.  Let's review some common scenarios that can be averted with some proper planning.

<!-- more -->

Common scenario's:
<ol>
<li>The time comes for a CDN and all images need to be served up with a URL like cdn.example.com.</li>
<li>The use of SSL is very common for authentication and other sensitive user information.  The problem is that SSL is much slower than a normal http request.  Traffic needs to be segregated by changing the SSL URL's from https://www.example.com to https://secure.example.com.</li>
</ol>

The solution to these problems is quite trivial: simply prepend a domain to a relative URL.  Consider the following config file:

<pre>
[development]
site.cdn = "http://dev.example.com"
site.ssl = "https://dev.example.com"

[production]
site.cdn = "http://cdn.example.com"
site.ssl = "https://secure.example.com"
</pre>

This configuration uses special URL values for a production environment, but uses the standard development server URL so the developers can still develop.  A simple addition to the php bootstrap file can set up defines to use in html templates.

<pre lang="php">
<?php
define('CDN', $config->site->cdn);
define('SSL', $config->site->ssl);
</pre>

And then in a .phtml file you can simply do the following:
<pre lang="html">
<img src="<?=CDN>/images/hero.png">

<a href="<?=SSL>/login.php">Login</a>
</pre>

Consider prepending the domain for all URL's in your application, not just those types listed above.  There are plenty of scenario's that may require AJAX calls or even normal GET/POST request to use different domains.
