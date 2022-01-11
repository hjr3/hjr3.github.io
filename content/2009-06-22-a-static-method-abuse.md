+++
title = "Static Method Abuse"
path = "/2009/06/22/a-static-method-abuse.html"

[taxonomies]
tags=["php"]
+++

When I began taking over the web development project at work, I noticed a developer using a lot of static members and methods in his class definitions.  His explanation was that it was an optimization he used to improve performance.  Unfortunately, he had no metrics to back the statement up.  So I set out to do some of my own.

<!-- more -->

The developer was using static methods and members in abstract classes.  Here is a very simple example:
<pre lang="php">abstract class Model_Abstract
{
    static $db;

    public static function setDb($db)
    {
        self::$db = $db;
    }
}</pre>
This was complimented by a call in the MVC bootstrap:
<pre lang="php">$db = new Db;
Model_Abstract::setDb($db);</pre>
The developer explained to me that this forces each concrete model class to use the <em>same</em> database class.  He said his reason for doing this was that PHP's copy-on-write functionality made a copy of the database object each time the internal results of that class changed.

I ran some tests (results below) to see if this was true and it, partially, was.  If any part of a class changes, PHP performs a shallow copy.  That is, if a database class contains a result class, only the result class will be copied and changed if a select query is performed.  The internal reference counter of the database class will simply be incremented.

I also tested whether static members and methods circumvented the shallow copy and they do.  However, this is not without confusion.  The results from the database class must be stored somewhere else before another query is made.  The next query will overwrite the previous results and cause some real confusion if someone forgets to do this.

This whole mess started as a code optimization technique.  It looked to like pre-optimization on code that was never that slow to begin with.  The developer thought that a deep copy of the class was being performed, but never actually bothered to check it out.  The amount of time and resources saved by using the static approach is infinitesimal that it is definitely not worth it.

I have a <a href="http://github.com/hradtke/Crimson/tree/master">small framework</a> that I intentionally <a href="http://github.com/hradtke/Crimson/blob/ed29837f2430fac1e001921ba9ee238daffade1d/Crimson/Model/Abstract.php">wrote this into</a>.  In a future post, I will be using this to outline the steps I used to refactor the code.
<h2>Tests</h2>
<strong>Non-static test:</strong>
<pre lang="php">class Result
{
    private $_result;

    public function setResult($result)
    {
        $this-&gt;_result = $result;
    }

}

class Db
{
    private $_sql;

    private $_result;

    public function __construct()
    {
        $this-&gt;_result = new Result;
    }

    public function query($sql)
    {
        $this-&gt;_sql = $sql;
        $this-&gt;_result-&gt;setResult(rand());
    }
}

class Model
{
    private $_db;

    public function __construct($db)
    {
        $this-&gt;_db = $db;
    }

    public function test()
    {
        $res = $this-&gt;_db-&gt;query('SELECT bar FROM foo');
        debug_zval_dump($this-&gt;_db);
    }
}

$db = new Db;

debug_zval_dump($db);

$model = array();

$before = memory_get_usage();
for($i = 0; $i &lt; 10; $i++) {     $model[$i] = new Model($db);     $model[$i]-&gt;test();
}
$after = memory_get_usage();

echo "Change in memory usage: ", $after - $before;</pre>
<pre>object(Db)#1 (2) refcount(2){
  ["_sql":"Db":private]=&gt;
  NULL refcount(2)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    NULL refcount(2)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(1866838064) refcount(1)
  }
}
object(Db)#1 (2) refcount(4){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(282674262) refcount(1)
  }
}
object(Db)#1 (2) refcount(5){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(415846557) refcount(1)
  }
}
object(Db)#1 (2) refcount(6){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(459928359) refcount(1)
  }
}
object(Db)#1 (2) refcount(7){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(46535217) refcount(1)
  }
}
object(Db)#1 (2) refcount(8){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(1038889971) refcount(1)
  }
}
object(Db)#1 (2) refcount(9){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(1206550183) refcount(1)
  }
}
object(Db)#1 (2) refcount(10){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(144575963) refcount(1)
  }
}
object(Db)#1 (2) refcount(11){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(1310083917) refcount(1)
  }
}
object(Db)#1 (2) refcount(12){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(1850865612) refcount(1)
  }
}
Change in memory usage: 4996</pre>
<strong>Static test:</strong>
<pre lang="php">class Result
{
    private $_result;

    public function setResult($result)
    {
        $this-&gt;_result = $result;
    }

}

class Db
{
    private $_sql;

    private $_result;

    public function __construct()
    {
        $this-&gt;_result = new Result;
    }

    public function query($sql)
    {
        $this-&gt;_sql = $sql;
        $this-&gt;_result-&gt;setResult(rand());
    }
}

class Model
{
    private static $_db;

    static public function setDb($db)
    {
        self::$_db = $db;
    }

    public function test()
    {
        $res = self::$_db-&gt;query('SELECT bar FROM foo');
        debug_zval_dump(self::$_db);
    }
}

$db = new Db;

debug_zval_dump($db);

Model::setDb($db);

debug_zval_dump($db);

$model = array();

$before = memory_get_usage();
for($i = 0; $i &lt; 10; $i++) {     $model[$i] = new Model();     $model[$i]-&gt;test();
}
$after = memory_get_usage();

echo "Change in memory usage: ", $after - $before;</pre>
<pre>object(Db)#1 (2) refcount(2){
  ["_sql":"Db":private]=&gt;
  NULL refcount(2)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    NULL refcount(2)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  NULL refcount(2)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    NULL refcount(2)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(1722572263) refcount(1)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(2057122892) refcount(1)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(332001950) refcount(1)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(508834141) refcount(1)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(519443361) refcount(1)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(1746125577) refcount(1)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(690789379) refcount(1)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(462332752) refcount(1)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(801369044) refcount(1)
  }
}
object(Db)#1 (2) refcount(3){
  ["_sql":"Db":private]=&gt;
  string(19) "SELECT bar FROM foo" refcount(1)
  ["_result":"Db":private]=&gt;
  object(Result)#2 (1) refcount(1){
    ["_result":"Result":private]=&gt;
    long(557914563) refcount(1)
  }
}
Change in memory usage: 4136</pre>
