+++
title = "Doing It \"Right\""
path = "/2009/09/06/doing-it-right.html"
+++

I have always been of the opinion that doing a task the correct way the first time is the best way to complete the task. I don't mean to imply that there is one correct way of completing a task, only that doing it correctly does not involve cutting corners or using hacks to complete the task.  I have always preached this ideal as a developer, never the decision maker.  Now, I am currently working on a project with a few colleagues of mine where I am a decision maker.  Earlier today I began wondering whether my ideals will change as this project grows.

<!-- more -->

To be clear, I do not think that doing it "right" necessarily means perfection.  For some aspects of development, perfection is impossible or not worth the resources required.  For example, I have never seen a large application with a 100% test coverage.  Some aspects of a program are harder to test than others and the cost may far outweigh the benefits.  I think test coverage of 80% is a very acceptable goal. Another example is the practice of writing a test for each resolved bug.  Again, the costs may outweigh the benefits for very trivial bugs.  For me, making a significant attempt at testing means it is being done "right" even if it is not perfect.

There are aspects of development that can be nearly perfect.  The coding standard, once decided upon, should be adhered to everywhere.  Tools, like PHP_CodeSniffer, inform the developer of any deviation from the standard.  This makes it trivial for any such deviation to be fixed <em>before</em> the code makes it into production. Yet many companies never enforce their coding standard. The fact that it takes minimal effort for the developer to write code that adheres to the standard, I think that doing it "right" means no violations are reported from PHP_CodeSniffer.

Doing it "right" means more than just high test coverage and code that adheres to a standard.  These are things that are easily measured and very visible.  Other aspects of development like design and code re-use are just as important. It is very easy to use the Registry Anti-pattern during development because it can get results fast.  Planning out proper dependency injection takes time, especially if one does not do it often.  Code refactoring to allow for code re-use is easily pushed off too because it is not immediately necessary.  Proper design and code re-use are long term investments in a project that pay off dividends.

At what point do the aforementioned ideals become secondary. If a paying client needs a significant modification to a project completed tomorrow, I may throw testing and proper design out the window to make the deadline if I am hurting for income.  All the design and re-use in the world are worth nothing if the project dies tomorrow.  However, if I find myself in that position then I think I made a mistake somewhere else along the way.

I hope to use this post as a litmus test to measure my growth in the industry.
