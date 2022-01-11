+++
title = "PHP Gearman Bootstrap Script"
path = "/2011/12/09/php-gearman-bootstrap-script.html"

[taxonomies]
tags=["gearman", "php"]
+++

Writing the scaffolding for gearman workers is a pretty trivial task using the pecl/gearman extension. Keeping that scaffolding consistent between all the gearman workers in your application can get tough. I created a script that will remove the boilerplate gearman code and allow gearman worker scrips to simply be function definitions.<!-- more -->The idea for gearboot first came to me when I was using GearmanManager to manage production gearman workers. The GearmanManager code assumes that the workers are simply function definitions. This makes it easy to write and organize the workers, but it makes common tasks like debugging workers painful as all output is hidden

In order to make development of the gearman workers easier, I created a script to startup the gearman worker in a similar manner to the GearmanManager. The major benefit is that any sort of error or exception is shown right on the terminal screen. No digging through logs wasting time tying to figure out what happened. If you have logging already setup, it is now trivial to add a stdout writer to the logger when running php in cli mode.

One of the biggest complaints I heard from fellow developers was that they had a hard time trying to figure out if the worker actually received the job from the client. I added a logging to stdout that signals when a job is received and what a job is finished.

I setup a pear channel to make installation real easy. For installation instructions and some examples check out the github page page: <a href="https://github.com/hradtke/gearboot" target="_blank">https://github.com/hradtke/gearboot</a>
