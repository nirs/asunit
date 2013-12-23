======
ASUnit
======
-----------------------------
AppleScript testing framework
-----------------------------

ASUnit is testing framework for AppleScript, influenced by SUnit, ASTest_ and Python unittest_ module.

Features:

* Easy to install, no dependencies, minimal boilerplate code
* Automatic and manual test case registration
* Run in *Script Editor*, display results in a new document 
* Aggregate multiple test scripts

This text is mostly a shameless copy of Kent Beck `SmallTalk testing framework`_, modified for ASUnit.

.. _ASTest: http://applemods.sourceforge.net/mods/Development/ASTest.php
.. _unittest : http://docs.python.org/lib/module-unittest.html
.. _`SmallTalk testing framework`: http://www.xprogramming.com/testfram.htm

Installation
============

To use ASUnit, unzip and copy the file `ASUnit.scpt` to the Scripts folder, either in your Library folder, or in the global Library folder in the startup disk.

To use in a script, your test script should inherit from ASUnit. In the example, ASUnit was installed in the global Library folder (local domain)::

    property parent : load script file ((path to scripts folder from local domain as string) & "ASUnit.scpt")

Cookbook
========

Here is a simple pattern system for writing tests. The patterns are:

=========== =====================================
Fixture     create a common test fixture
Test Case   create the stimulus for a test case
Check       check the response for a test case
Test Suite  aggregate Test Cases
Test Runner run a test suite and display results
Test Loader aggregate test scripts
=========== =====================================

Fixture
-------

A fixture is certain configuration of the system, required to test certain feature. A complete tess for community of objects will have many fixtures, each of which will be tested many ways. 

Design a test fixture
~~~~~~~~~~~~~~~~~~~~~

1. Create fixture script, inheriting from TestCase
2. Add properties for each known object in the fixture
3. Override setUp to initialize the properties 

AppleScript lack introspection, so registering  of fixture is done by using factory methods during compile time. Instead of inheriting from TestCase directly, call ``makeFixture()``, which register the fixture and make your script inherit from TestCase.

In the example, the test fixture is two lists, one empty and one with elements. First create a script object, then add properties for the objects we need to reference later::

    script |accessing list|
        property parent : makeFixture(me)
        property empty : missing value
        property notEmpty : missing value

Then override setUp to create the objects for the fixture::

    on setUp()
        set empty to {}
        set notEmpty to {"foo", 1}
    end

	
Test Case
---------

You have a Fixture, what do you do next?

How do you represent a single unit of testing?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each test of the fixture is represented by a script object in the fixture. Test scripts uses the fixture setUp and tearDown handlers to make sure different tests do not interfere with each other.

Represent a predictable reaction of a fixture as a script object
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Add a script object inheriting from the fixture
2. Stimulate the fixture in the ``test()`` method

Again, you don't inherit directly from the fixture, but call ``registerTestCase()``, which register the test case and make it inherit from the current fixture.

We can predict that adding "bar" to an empty list will result in "bar" being in the list. Add a script object to the fixture, and stimulate the fixture in its run handler::

    script |add item|
        property parent : registerTestCase(me)
        set end of empty to "bar"
        ...
    end
    
Once you have stimulate the fixture, you need to add a Check to make sure your prediction came true.

Check
-----

A Test Case stimulates a Fixture.

How do you test for expected results?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you’re testing interactively, you check for expected results directly in the results or event log pane. You want a way to programmatically look for problems. One way to accomplish this is to use the standard error handling mechanism with testing logic to signal errors::

    if empty does not contain "bar" then error "where is bar?!"
    
When you’re testing, you’d like to distinguish between errors you are checking for, like getting six as the sum of two and three, and errors you didn’t anticipate, like subscripts being out of bounds or messages not being understood.

When a catastrophic error occurs, the framework stops running the test case, records the error, and runs the next test case. Since each test case has its own fixture, the error in the previous case will not affect the next.

The testing framework makes checking for expected values simple by providing a method, ``should()``, that takes an expression as first argument, and an error message as second argument. If the expression evaluates to true, everything is fine. Otherwise, the test case stops running, the failure is recorded with the error message, and the next test case runs.

Check by calling should() with an expression
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the example, after stimulating the fixture by adding "bar" to an empty list, we want to check and make sure it's in there::

    script |add item|
        property parent : registerTestCase(me)
        set end of empty to "bar"
        should(empty contains "bar", "no bar?!")
    end

There is a variant on ``should()``. ``shouldnt()`` causes the test case to fail if the expression argument evaluates to true. It is there so you don’t have to use "not (...)".    

Once you have a test case, you can run it. Send ``test()`` to the script object::

    |accessing list|'s |add item|'s test()
    
If it runs to completion, the test worked. If you get an error, something went wrong. The result of sending ``test()`` to a TestCase is a TestResult object.


Test Suite
----------

You have several Test Cases.

How do you run lots of tests?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Soon you will have many test cases, and fixtures. You could just string together a bunch of expressions to run test cases. However, when you then wanted to run "this bunch of cases and that bunch of cases" you’d be stuck.

The testing framework provides an object to represent "a bunch of tests", TestSuite. A TestSuite runs a collection of test cases and reports their results all at once. TestSuites can also contain other TestSuites, so you can put Joe’s tests and Tammy’s tests together by creating a higher level suite.

Combine test cases into a test suite
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Tests are automatically collected in each script file. To make this happen, you must create a property named *suite* and initialize it with a TestSuite in each ASUnit test script. As usual, you use a factory function for that.

Add this code at the start of your test script using a descriptive name for the test suite::

    property suite : makeTestSuite("Test Suite Name")
    
To run all the tests in the the current file, tell suite to test::

    set results to suite's test()
    
The result of sending "test" to a TestSuite is a TestResult object. It records all the test cases including their failures or errors, and the time at which the suite was run.


Test Runner
-----------

Running a TestSuite and inspecting the TestResult returned is not very convenient. You would like to click one button to run a suite of tests. The framework supply a TextTestRunner, that run a TestSuite and display progress and test results.

To run the suite in a test script, create a TextTestRunner and tell it to run::

    run makeTextTestRunner(suite)

Now when you click the *Run* button in *Script Editor*, the test will run and you will get a test report in a new *Script Editor* document.


Test Loader
-----------

Soon you will have more than one test script. It is too much work to open each and run the tests each time. The framework supply a TestLoader, which search for tests scripts in folders, and return a suite with all the tests. TestLoader looks for file names that starts with *Test* and ends with *.scpt*, like *Test Module.scpt*, in the folder you specify.

To collect and run all the test scripts in the current directory, create this script::

    property currentFolder : folder of file (document 1's path as POSIX file) of application "Finder"    property parent : load script file ((path to scripts folder from local domain as string) & "ASUnit.scpt")  
    set suite to makeTestLoader()'s loadTestsFromFolder(currentFolder)    
    run makeTextTestRunner(suite)

Now you may open a single script and click the *Run* button to run all your test scripts.


Example Test Script
===================

Here is an example test script you can use as a template for your scripts::

    property parent : load script file ((path to scripts folder from local domain as string) & "ASUnit.scpt")
    property suite : makeTestSuite("My Tests")
        
    script |accessing list|
        property parent : makeFixture(me)
        property empty : missing value
        property notEmpty : missing value
        
        on setUp()
            set empty to {}
            set notEmpty to {"foo", 1}
        end setUp
        
        script |add item|
            property parent : registerTestCase(me)
            set end of empty to "bar"
            should(empty contains "bar", "no bar?!")
        end script
        
        script |add same item|
            property parent : registerTestCase(me)
            set end of notEmpty to "foo"
            should(last item of notEmpty is "foo", "first foo vanished?!")
            should(first item of notEmpty is "foo", "where is last foo?!")
        end script
    end script
    
    run makeTextTestRunner(suite)


License
=======

:copyright: © 2006 Nir Soffer
:license: GNU GPL, see COPYING for details





