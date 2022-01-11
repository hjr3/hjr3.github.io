+++
title = "Phing - phplint task"
path = "/2009/11/18/phing-phplint-task.html"

[taxonomies]
tags=["build", "phing", "php"]
+++

<a href="http://phing.info/trac/">Phing</a> is a PHP port of Java Ant.  It is a great tool to use in development.  It standardizes a lot of build scripts you would have to maintain internally.  Unfortunately, examples seem to be lacking.  As a quick introduction to Phing, I will show how you can check all your php scripts for syntax errors.

<!-- more --> If you have never heard of Phing before, I suggest you visit the <a href="http://phing.info/docs/guide/current/chapters/GettingStarted.html">getting started page</a>.

The phplint task will syntax check one or more files containing php source code.  All it requires is a list of files.

Here is the sample build.xml file:
<pre lang="xml">
<?xml version="1.0"?>
<project name="MyPhpProject" basedir="." default="lint">
    <target name="lint">
        <phplint>
            <fileset dir=".">
                <include name="*.php" />
                <include name="**/*.php" />
            </fileset>
        </phplint>
    </target>
</project>
</pre>

We created a target called "lint".  This will allow us to call lint from the command line.  In that target we specify a single task: phplint.  The phplint task will call "php -l" on each filename provided.  Within that task we specify a <a href="http://phing.info/docs/guide/current/chapters/appendixes/AppendixD-CoreTypes.html#Fileset">fileset</a> type.  The fileset type is what provides the phplint task with files to syntax check.

The fileset type can get pretty complex.  I have intentionally kept it simple.  The dir attribute on the fileset type specifies the current directory.  The first include element will match any file with a ".php" extension in the current directory.  The second include element will match any file with a ".php" extension in a sub-directory.

Here is a visualization of how the fileset is working:
<pre>
app/
    build.xml <-- will not match
    foo.php <-- will match
    bar.php <-- will match
    includes/
        baz.php <-- will match
        special/
            special1.php <-- will match
            special2.inc <-- will not match
    sql/
        schema.sql <-- will not match
</pre>

Place the build.xml file at the root of your applications directory structure.  You can move it to another folder after you get more comfortable with phing.

From the command line run: phing lint

<img alt="" src="http://farm3.static.flickr.com/2648/4116616256_327cf8fb1a.jpg" title="phplint task" class="alignnone" width="500" height="196" />
