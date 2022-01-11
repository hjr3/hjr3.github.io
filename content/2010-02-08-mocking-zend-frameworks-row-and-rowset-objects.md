+++
title = "Mocking Zend Framework's Row and Rowset objects"
path = "/2010/02/08/mocking-zend-frameworks-row-and-rowset-objects.html"

[taxonomies]
tags=["php", "phpunit", "unittest", "zendframework"]
+++

If you separate your business logic from your data access logic, the last thing you want to do is make your business logic unit tests reliant on the database.  This is normally not a big deal: retrieve the data, store it in an array and pass it off to the class with the business logic.  Mocking the data for the unit test simply requires you to hardcode from array information in the test.  However, I recently ran into a case where I wanted to pass Zend_Db_Table_Row and Zend_Db_Table_Row objects to the business logic and mocking them was not so easy.

<!-- more -->I first attempted to mock Zend_Db_Table_Row using PHPUnit's _getMock method.  This proved to be an exercise in futility.  I did not want the class to connect to the database to verify whether or not the columns were valid.  After a few frustrating hours, I started to wonder how Zend was unit testing Zend_Db_Table_Row.  So I downloaded the full version of the latest Zend Framwork and started poking around.  I stumbled upon My_ZendDbTable_Row_TestMockRow hiding away in ZendFramework-1.9.5/tests/Zend/Db/Table/_files/My/ZendDbTable/Row/TestMockRow.php.  I will not go into what was done to make it a usable mock, but I will show you how to use it.

<pre lang="php">
<?php

require_once 'PHPUnit/Framework/TestCase.php';
require_once 'tests/Zend/Db/Table/_files/My/ZendDbTable/Row/TestMockRow.php';

class TestMockRowTest extends PHPUnit_Framework_TestCase
{
    public function testRowHasIdValue()
    {
        $data = array(
            'data' => array(
                'first_name' => 'Herman',
                'last_name' => 'Radtke',
                'email' => 'herman@example.com'
            )
        );

        $row = new My_ZendDbTable_Row_TestMockRow($data);
        $this->assertEquals('Herman', $row->first_name);
        $this->assertEquals('Radtke', $row->last_name);
        $this->assertEquals('herman@example.com', $row->email);
    }
}
</pre>

Creating a mock row object is much like hardcoding an array.   Define an array that has a single key 'data' that contains an array as a value.   Inside this array, the database column name is the array key and the database column value is the array value.

That class works great for mocking a single row object, but I still needed a solution for multiple row objects.  I expected to find a class similar to My_ZendDbTable_Row_TestMockRow for the purposes of testing Zend_Db_Table_Rowset, but none existed.  Fortunately, it took only a few minutes to create my own.  All one has to do is specify the name of the row class for the rowset class to use.

<pre lang="php">
<?php

require_once 'PHPUnit/Framework/TestCase.php';
require_once 'library/Zend/Db/Table/Rowset/Abstract.php';
require_once 'tests/Zend/Db/Table/_files/My/ZendDbTable/Row/TestMockRow.php';

class ZendDbTableMockRowset extends Zend_Db_Table_Rowset_Abstract
{
    protected $_rowClass = 'My_ZendDbTable_Row_TestMockRow';
}

class RowsetTest extends PHPUnit_Framework_TestCase
{
    public function testConstructor()
    {
        $rowset = new ZendDbTableMockRowset(array(
            'data'=> array(
                array(
                    'id' => 123456
                ),
                array(
                    'id' => 123457
                ),
                array(
                    'id' => 123458
                ),
                array(
                    'id' => 123459
                ),
                array(
                    'id' => 123460
                )
            )
        ));

        $id = 123456;
        foreach ($rowset as $row) {
            $this->assertTrue(($row instanceof My_ZendDbTable_Row_TestMockRow));
            $this->assertEquals($id, $row->id);
            $id++;
        }
    }
}
</pre>

The array passed to the mock rowset has a similar structure to the one we used above.   The only difference is that we have multiple arrays, inside the 'data' array, each representing one row.   For testing purposes, I created an 'id' field.   I normally would not ever use an artificial key field inside a business logic class since that value is very dependent on the database.

I created a ZendDbTableMockRowset class in my projects test/mocks directory so I can use it in multiple test files.  I also copied My_ZendDbTable_Row_TestMockRow into the test/mocks directory so I would not be dependent on the external tests from Zend Framework.  Now mocking Row or Rowset objects is just as fast as mocking arrays.
