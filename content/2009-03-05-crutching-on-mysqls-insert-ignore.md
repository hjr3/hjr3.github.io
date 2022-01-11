+++
title = "Crutching on MySQL's INSERT IGNORE"
path = "/2009/03/05/crutching-on-mysqls-insert-ignore.html"

[taxonomies]
tags=["mysql", "postgresql", "SQL"]
+++

MySQL has a lot of nice additions to the SQL-92 standard that make life easier for developers.  The drawback to these shortcuts is that many developers don't learn how to cope in other database environments, like PostgreSQL.  Take INSERT IGNORE for example:  PostgreSQL does not have support for this syntax.  A resourceful developer can do a quick google search to learn an alternative, but many of the solutions at the top of google will show the wrong way to solve this problem.

<!-- more -->

Common bad example 1: Use a SELECT statement to check if the key exists and INSERT only if the key is not found.  
The main problem with this solution is the glaring race condition.  In the time in between the SELECT and INSERT, another session can insert the record.  You also have to write extra code into your application to process the SELECT query, which may have to specific for many cases.  A big waste of time.

Common bad example 2: Delete the row containing the key to be inserted and then INSERT the data again.  
Hopefully this is done in a transaction, but it still is dicey.  While this solution requires no extra application code to be written, it will only work for very basic cases.  Timestamps, flags and other meta data will be lost with a blind delete.  This solution scares me the most.

Both examples 1 and 2 can also be made into stored procedures.  Don't be tricked into thinking this is a better solution.

<strong>The best solution</strong> is to insert the data and, if there is an error, check the error code.  This is basically what the MySQL INSERT IGNORE syntax does!  PostgreSQL, and most major relational databases, follow the ANSI SQL-92 standard.  The SQL-92 standard contains a clear list of error codes that an application can check against.  If there is a duplicate key error, the application can handle it accordingly.  This does not require an extra query, has no race condition and is easy to check for in the application.

For those wondering, 23505 is the error code for duplicate key errors.

A list of all the SQL error codes can be found here:
<a href="http://db.apache.org/derby/docs/10.3/ref/rrefexcept71493.html">http://db.apache.org/derby/docs/10.3/ref/rrefexcept71493.html</a>

Edit: Maggie Nelson talks about her experience with this very same issue from an Oracle users point of view in <a href="http://maggienelson.com/2009/03/the-rules-of-software-engagement/">The Rules of (Software) Engagement</a>.
