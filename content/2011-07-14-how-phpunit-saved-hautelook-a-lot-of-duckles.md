+++
title = "How PHPUnit saved HauteLook a lot of duckles"
path = "/2011/07/14/how-phpunit-saved-hautelook-a-lot-of-duckles.html"

[taxonomies]
tags=["hudson", "php", "phpunit", "unittest"]
+++

<p style="text-align: left;">Yesterday I tweeted that a phpunit test prevented a very expensive error at <a href="http://www.hautelook.com/">HauteLook</a>. I had shared the details internally, but after receiving some positive twitter feedback I decided to make the story public. I am of the opinion that unit tests are a worthwhile investment. Knowing full-well that some people are very skeptical of unit tests, I think it is important to have real world examples of unit tests paying valuable dividends on that investment.<!-- more --></p>
<p style="text-align: center;"><a href="https://twitter.com/#!/hermanradtke/status/91300142115848192"><img class=" aligncenter" title="PHPUnit tweet" src="http://farm7.static.flickr.com/6135/5937415462_af24107870.jpg" alt="A #phpunit test just prevented a very expensive error. This is a perfect real world example for my next unit testing presentation." width="492" height="213" /></a></p>
A few days ago Hudson began emailing alerts that a freight calculation unit test was failing in production. I emailed the developers responsible for the code and asked them to investigate this right away. The funny thing about unit tests failing in production is that developers can fall into a mindset of "the code is working on production, so there must not be a problem". The freight calculations did appear to be working correctly on production. Because we are so busy, no further investigation was done until yesterday.

Yesterday afternoon two developers realized that this failing unit test was the result of a serious regression in our "premium" freight calculation. This freight calculation type is most commonly used for furniture items and can easily calculate a shipping cost well over $100. This particular bug was calculating the "premium" freight costs as free. Now HauteLook's business model is flash sales. This means new sales events start at 8:00 am PT each morning and most of the business is done within the first few hours of the event. If there had been a furniture sale event within the past couple of days, HauteLook would have given away thousands of dollars worth of free shipping (or even tens of thousands if the event was large enough) before we would have noticed and fixed the issue. Shipping costs were not the only thing we saved either. The amount of  time, energy and stress that it would have taken for the various  departments to address this issue would have been high.

I saw an immediate change in the way my colleagues view unit tests. This was a clear indication that our investment has paid off. In this case we had well written unit tests that quickly alerted us to a problem before it costs us anything. I know next time a unit tests fails on production, HauteLook developers will not assume everything is working.
