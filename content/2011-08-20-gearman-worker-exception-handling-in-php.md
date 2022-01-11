+++
title = "Gearman Worker Exception Handling in PHP"
path = "/2011/08/20/gearman-worker-exception-handling-in-php.html"

[taxonomies]
tags=["asynchronous", "gearman", "lambda", "php"]
+++

Gearman is one of my favorite technologies to use. So much in fact that I recently decided to take over the maintenance of pecl/gearman. While asynchronous tasks are a great feature, I find the ability to run multiple tasks in parallel to be much more useful. One of the biggest shortcomings of this approach was that uncaught worker exceptions would be treated as a successful completion of a job. I used to wrap all my workers in a generic try/catch block to prevent this from happening.  With the latest commits to pecl/gearman, I can now use the exception callback to properly track the exceptions.

<!-- more -->I use Gearman for most of my batch processing problems. The process is simple: create any number of tasks using GearmanClient::addTask() and then run the tasks in parallel with GearmanClient::runTasks(). All available workers that can handle that task will be used. In order to keep track of the batch process, I define callback functions in GearmanClient::setCompleteCallback() and GearmanClient::setFailCallback().

<script type="text/javascript" src="https://gist.github.com/1159750.js?file=gearman-client-batch-example.php"></script>

<p>Every once in a while one of the workers would throw an exception that would not be caught. One would expect that this would trigger GearmanClient::setExceptionCallback(). Until recently, pecl/gearman was not properly handling exceptions from workers. This might not be so bad if an uncaught exception sent back a status of GEARMAN_WORK_FAIL and triggered the fail callback. The problem was that an uncaught worker exception actually sent back GEARMAN_SUCCESS. This made it appear to the client that the tasks was successfully completed. The workaround at the time was to wrap all workers in a generic try/catch block to prevent any uncaught exceptions. The catch block would then need to explicitly send back a status of GEARMAN_WORK_FAIL. This caused a lot of boilerplate code and each developer had to become intimately aware of how fickle pecl/gearman was with exceptions.</p> 

<p>I decided to fix pecl/gearman to properly trigger the exception callback in the event of an uncaught exception. The worker would still die, but now the client would be properly informed. This has to be done in two parts. I had to first update the pecl extension to detect when an exception occurred. The second part was sending back the proper status to the Gearman daemon so the client could be informed. This was a little more tricky than it first appeared. The Gearman daemon does not automatically handle the exception status. Both the client and the worker have to tell the Gearman daemon that they want to enable exceptions when they connect. Once I figured out how to do this using libgearman I was able to trigger the exception callback.</p> 

<p>This exposed another issue with the way libgearman works. Because the exception handling is optional, the worker will send back a status of GEARMAN_WORK_FAIL even a status of GEARMAN_WORK_EXCEPTION was already sent. This means that the fail callback can be triggered after the exception callback. You can have the exception callback return a status of GEARMAN_WORK_EXCEPTION, but that will cause the entire batch process to stop. I am working with Brian Aker, who maintains libgearman, to see if something can be changed. Until then, I am required to keep track of tasks that threw an exception and check that list in the fail callback.</p>

<script type="text/javascript" src="https://gist.github.com/1159694.js?file=gearman-client-exception-callback.php"></script>
<script type="text/javascript" src="https://gist.github.com/1159700.js?file=gearman-worker-exception.php"></script>
