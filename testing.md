h1. Testing 101

Unit tests are a key way to stay sane in Python land.  The flexibility and speed of Python make it fun.  They also make it hard to be sure you have not broken things by accident.  Since we don't get compile-time guarantees about the quality of our code, it's important to embrace testing as a way to guarantee that things are behaving as we expect from one revision to the next.

## What Are Unit Tests?

Unit tests are only one of many ways to test code.  They get the name 'unit' tests to remind us what they do -- and what they don't do. 

Unit tests test small 'units' of code: typically, individual functions and classes. Their job is to provide the guarantees that Python cannot -- that, for example, when you call `give_me_a_string()` you get back a string, and not a number, or that when you call `agate_project_location()` you get back a file path with right-slashes. The essence of the pattern is "I expect this return value from this call."  The key to testing is to guarantee that **the same function called with the same arguments returns the same thing every time.**   That's obviously a less ambitious goal than demonstrating that "everything works"  -- there will always be combinations of factors that have not been tested and which can surprise you.  However it's extremely difficult to tackle those larger, systematic issues when the building blocks are not solid and reliable.  **Unit tests are there to guarantee that the building blocks are in reliable**. By nailing down the pieces, you can add stability to larger and larger parts of the system.

Essentially, tests are *promises*.  They're a way of telling users and readers that the code is going to behave a certain way.  In a lot of ways testing is the answer to the old programmer's complaint that "comments don't compile".   Tests are comments with teeth.

[This article](https://realpython.com/python-testing/) is a great introduction to Python testing, although you can skip the sections relating to `nose`, `django` and `tox` which we don't use.

## Do You Have To Write Tests?

**Yes.**  We have a department level OKR that more than half of all our new code will be tested to some degree by the beginning of Q4 2019.  Tests should become a regular part of you python workflow.

A lot of advice writers will tell you that you should [write the tests first](http://pythontesting.net/agile/test-first-programming/).  As of today this is not a requirement -- however it's an interesting idea and can be very useful for some kinds of work.  It tends to work better for purely algorithmic code and not as well for things like Maya programming where a lot of work requires messing around with somebody else's API.


# How to write tests

We use the builtin `unittest` module for testing.  

The key class in unittest is `TestCase`, which is a collection of tests that can be run together. Every function on a TestCase object whose name begins with `test_` will be called by the test runner software.  Inside that function we use the built-in `assert` to demonstrate that a condition is true (or [truthy](https://medium.com/@dawranliou/python-truth-value-testing-is-awesome-dae6c23cc1c2))  If the assert passes, then so does the test. 

Here's a very simple unittest example to test a hypothetical function `myfunction` in a module called `mymodule`

```
import unittest
from mymodule import myfunction

class TestMyFunction(unittest.TestCase):
 
    def test_returns_string(self):
        assert isinstance(myfunction(), str)

    def test_result_is_lowercase(self):
        result = myfunction()
        assert result == result.lower()

    def test_result_contains_no_numbers(self):
        result = myfunction()
        for num in '0123456789':
            assert num not in result
```

This example seems trivial, but it's a clear example of the fact that **tests are documentation that compiles**, which is one the reasons they are so important. Reading the test demonstrates that:

1. `myfunction()` has to return a string
2. `myfunction()` returns lowercase letters
3. `myfunction()` will not return any numbers

A potential customer should feel confident calling `myfunction` will return a lower-cased string with no numbers in it.

However, his example also illustrates a potential problem: these tests, useful as they are, don't tell us _everything_ we might want to know.  For example, what about spaces or punctation characters? How long is the return value supposed to be? Can we expect some other patterns in the results?  

The more comprehensive the test suite, the more assumptions we can make about how the function is supposed to be used.

## Setup and Teardown

Sometimes its important to set some preconditions for the tests.  For example, if you're testing a function that does something in a maya scene, you might want to set up you tests so that they creat some test geometry before every test.  The `TestCase()`  class offers two functions to help with this.

`setUp()` is called before every test in the TestCase is run.
`tearDown()` is called after each test completes.

Here's an example of using `setUp()` to do some Maya related testing:

```
import unittest
import maya.cmds as cmds
import my_maya_tools

class TestMyMayaTools(unittest.TestCase):
    def setUp(self):
        # set up every test with a polyCube and a polySphere
        cmds.file(n=True, f=True)
        cmds.polyCube()
        cmds.polySphere()

    def test_find_cube(self):
        assert my_maya_tools.find_cubes() == ['pCube1']

    def test_find_sphere(self):
        assert my_maya_tools.find_spheres() == ['pSphere1']

    def test_no_sphere_after_deletion():
        cmds.delete('pSphere1')
        assert my_maya_tools.find_spheres() == []
```

And here's an example where you'd use `tearDown()` to reset after testing a module that provides a counter:

```
import unittest
import counter

class TestMyMayaTools(unittest.TestCase):
    def tearDown(self):
        counter.reset(0)  # reset the counter after each test

    def test_increment_one(self):
        counter.increment(1)
        assert counter.value() == 1

    def test_increment_ten(self):
        counter.increment(10)
        assert counter.value() == 10

    def test_increment_multiple_times(self):
        counter.increment(5)
        counter.increment(6) 
        assert counter.value() == 11
```


# Structuring Tests For Your Projects

Typically a project -- whether it's the maya tool set or a product like launchpad -- will contain a tests folder.  In order to simplify test discover, you want to set up your test folder as follows:

* Include an `__init.py__` in the test folder so it looks like a Python package.  Python has an easier time finding tests that are set up this way.  If your tests require any special setup (for example, adding things to the path, or making sure that maya tests are run after `maya.standalone.initialize()` has been called) do it in the `__init.py__`
* Inside the test folder, create a test files called `test_<some-name-here>.py` Typically you want the name of the file to correspond to the name of the module or package you are testing.  So the `RigTool` package should be `test_RigTool.py` and so on.  When running tests, you can choose to run only some files and not others.
* In the test file, import `unittest.TestCase` and subclass it to create test suites as in the example above.  You'll typically want to create classes that have similar setup and teardown conditions.  Every method inside your `TestCase` subclass that begins with `test_` will be found and run by the test runners.

In the Maya tools, we've already got the test folder and some additional helper code that assists with things like maya-specific initialization.  See the `ul_tests` folder and in particular the comments to `ul_tests/__main__.py`

# Running tests

The completely vanilla python way to run tests is to somewhere where your test module is importable (typically, that means being in the same folder where your test folder lives) and then calling 

```
python -m unittest discover your_test_module
```

Because there are usually issues with path setups and so on, we have two slightly custom methods to run our tests.

## Running tests in Maya

Maya tests have to be run with Mayapy, rather than regular python.  So to run tests:

   1. cd into the ulMaya directory
   2. mayapy.exe -m ul_tests 

(Make sure you are running the mayapy.exe that comes with our currently supported Maya!) 

You can save time and focus on just what you want to check by specifying the test modules you want to run. For example, to run the tests in `ul_tests/test_hash.py` you'd do this:

   1. cd into the ulMaya directory
   2. python -m ul_tests test_hash

(Note that you're specifying module names, not filenames -- **don't** use the 'py' extension)

The code in `ul_tests/__main__.py` is a good starting place if you have to create some kind of custom test rig for yourself.

## Running tests for other products

When working on a different project such as, say, the `agate_content_check` project you'll want to set your tests up in a folder called `tests` at the root of the project.  Inside that folder add the `__init__.py` as above (it can be blank if there's no special setup.  Write your tests in that folder).

You can run the tests manually by activating your product virtualenv and then running them using `python -m unittest discover`, but most of our products are stuctured with extra folders that may mess with your python path.  So, the easiest way to handle that is to add a test path to the `setup.py` for your project that points at your test directory (the path(s) you supply are assumed to be relative to the location of `setup.py`). 

For a product that looked like this:

     product
        |------ packages
        |          |------ my_module.py
        |
        |------ tests
        |          |------ __init__.py
        |          |------ test_my_module.py
        |
        |------ setup.py

Your setup.py would look like this:

```
    from setuptools import setup, find_packages

    setup(name='product',
          package_dir={"": "packages"},
          packages=find_packages('packages'),
          package_data={"": ["*.*"]},
          tests='tests')
```

Then you can run the tests with 

`python setup.py test`

Although you can run the tests other ways, products that are going to be included in the studio-python distribution should be set up this way.  The `publish` script we use to update the python environment expects this setup.


# Writing good tests

Here are some of the important virtues you want to build into your tests.

## Reliable

The most important thing any test does is to guarantee repeatability. If the test is `one_plus_one_equals_two()`  then every single time it runs it should produce the same result. There's no room for randomness in tests.

## Simple

A good test is very simple.  It focuses on one -- only one -- behavior and makes sure it's working.  If there are other things to worry about, then they should get separate tests.  That helps pinpoint where problems really occur.  For example if you have a class that is supposed to be giving you information about a path on disk, you might care about several very different things:

    * Does the class produce properly formatted paths, with the kind of slashes you expect?
    * Does the class handle the upper and lower case values in a predictable way?
    * Does the class produce useful errors when fed bad data?
    * Does the class actually give back paths which exist on disk?

In practice, you'd want all of these things to be correct when using your class. But in testing, you want to isolate these behaviors and test each one separately. That will help you understand emergent bugs in the simplest way possible.  As you discover other problems along the way, you can add new tests to shore up the stability of the class while not compromising existing functionality: for example if you discover your path class needs to handle paths with UNC addresses as well as disk paths, you can add tests to support UNC while remaining confident that you have not accidentally changed the way your code was running in other ways.

## Short

Good test code is typically short and sweet.  Individual tests are often only a couple of lines.  If test code starts getting beyond a dozen lines, that may be a sign that the code is too complicated: if the setup is so complex that it's a hassle to test, that may mean you've got too much functionality clustering together.

## Isolated

Good tests exist in isolation.  Ideally they don't need you to set up files on disk or set environment variables or do other kinds of setup tasks.  

Obviously there are aspects of tasks you'll want to do that do depend on things like files on disk, or network access, or the state of a scenee in Maya. It's a good idea to design your code so the moving parts and the configuration-specific parts are separated out.  For example, here are two ways of writing the same function:

```
    def file_parser(filename):
        with open(filename, 'rt') as file_handle:
            return [line for line in file_handle if 'x' in line]
    
    # example: print my_file_reader('test.txt')

    def decoupled_parser(filehandle):
        return [line for line in file_handle if 'x' in line]
    
    # pass in a file handle object: 
    #     with open('test.txt', 'rt') as fh:
    #         print decoupled_parser(fh) 
    # or use a substitute for easy testing.
    #     test = StringIO("x\ny\nz\nxx\n")
    #     test.seek(0)
    #     print decoupled_parser(test)
```

These examples are functionally identical -- but the second version doesn't impose any assumptions about the state of the user's disk.  It's much easier to test (because you don't need to write a file just to read it, and you don't have to worry about things like drive letters or permissions).  It's also much faster, because it doesn't involve disk access.

Sometimes you can't avoid being dependent on code that's out of your control -- for example, when you want to talk to a database or run a command in Maya.  The `mock` module, below, is a great tool for creating tests around code that's not natively well suited for testing.  Howeve decoupling IO, network operations, and things like that from from logic will make for better code overall -- mock should be for special cases.

## Up-to-date

One of the most important things we get from tests is a strong guarantee of what the public api of our work looks like.  Try to make sure that **every public function or class in your module gets at least a cursory test**.  Even a completely simple "does this exist at all" test has value in maintaining the public api:

    def test_my_function_exists(self):
        assert callable(my_function)

This is not a very fancy test but it guarantees that `my_function` actually exists and is a callable function.  If, for example, you accidentally typed an extra character into the name of the function you might only find out about it at runtime -- unless you have a test like that that guarantees the function is there.

The corollary to this is that you should **remove tests that are no longer needed** -- if you deliberately delete `my_function` your should also remove `test_my_function_exists`.

It's possible to turn off tests individually when you're working.  However it's not a good idea.  You should never knowingly check in code with broken tests -- either fix the code (better) or remove the test if the test is no longer predicting the behavior of the code correctly.

# What to test

The goal of a unit test is to test **small, identifiable** aspects of a system. It's _not_ to test an entire system at once.  

Test guru's frequently talk about a "testing pyramid".  The base of the pyramid is unit tests, which guarantee the stability of components used by larger system.  This consitutes two thirds to three quarters of the test coverage for a sytem.  The middle of the pyramid is "API" or "integration" tests -- tests that guarantee the pieces come together correctly and that the entire environment actually functions.  In our universe, this is handled primarily by the build server's compilation tests, which help ensure that modules actually exist, are importable, and don't include syntax errors.  The tip of the pyramid is "system" testing, which is driven by actual users.  In our terms, that's opening Maya or Launchpad and seeing it work as expected. 

![](https://dzone.com/storage/temp/6496415-testingpyramid.png)

The takeaway is that unit tests are not the place where you try to guarantee a particular user experience. The rule of thumb is that unit tests **test the building blocks, not the building**. Library and framework code that is used extensively by other code should be heavily tested -- both because this guarantees reliability in the field and because subtle changes can be interpreted differently by the many other tools that consume this code.  The flip side of this is that user-facing tools -- especially UI -- don't usually work well for unit tests.  They need human attention. 

So, unit tests are vital for library code that is designed to do discreet jobs in a reliable way.  For example a module that creates an octree is an ideal candidate for tests, because it's got a single, well-defined tasks and can easily be tested in isolation.   On the other hand a tool which involves user interaction and then performs a lot of complex operations on a Maya scene is extremely hard to test: a successful 'run' of the tool requires many conditions, from the state of the file to the intent of the user, which are hard to simulate in a test environment.  Unit tests contribute to the stability of tools like this indirectly, by making sure that the smaller bits of code which the tool relies on are depenable -- but they don't guarantee that a particular button does what you think it does.


# Advanded testing topics

## Mocking

Functions that rely on the outside world are the hardest to test.  For example, you might have a function that's expected to read and then update a file on the user's hard drive.  If you actually run the function on your local machine, you will have a hard time knowing that the function is doing what you expect, because your own machine could have bad data.  You might also not like the fact that running the tests messes with your local files -- or that tests fail on the build server because a file is missing.

Python includes a very powerful test helper module called `mock`, which allows you to substitute dummy objects and functions. Mock allows you to verify that functions are being called with the arguments you expect, or to _pretend_ that a function call or object has returned a value you want to test.

For example, this maya function by itself is very difficult to test, because it depends on the state of Maya at a moment in time.

```
def is_dayton_file():
    filename = cmds.file(q=True, sn=True)
    return '/UE4/DaytonGame/Artsrc/' in filename
```

you can use the mock module to temporarily replace the call to `cmds.file` and get it to return a known value that is testable, without having to actually save a maya file on disk:

```
import unittest
import mock  # note: in python3 this is 'import unittest.mock'
import maya.cmds as cmds
from my_module import is_dayton_file

class TestDaytonFile(unittest.TestCase):
    def test_is_dayton_positive(self):
        with mock.patch('cmds.file', return_value = 'c:/ul/main/UE4/DaytonGame/Artsrc/Test.ma'):
        assert is_dayton_file()

    def test_is_dayton_negative(self):
        with mock.patch('cmds.file', return_value = 'c:/some/other/path/Test.ma'):
        assert not is_dayton_file()
```

You can even mocks to simulate file reads and writes. This is a tough method to test because it hides a file call inside itself:

```
def get_users_from_file(filename):
    with open(filename, 'rt') as file_handle:
        values = json.load(file_handle)
        return [item['user'] for item in values if item.get('active')]
```

With `mock` you can test the logic without messing with your disk state by doing something like this

```
import unittest, mock
class TestFileParser(unittest.TestCase):

    def test_get_users_from_file (self):
        DUMMY_DATA = "[{'user': 'steve'}, {'user':'matt', 'active':true}]"
        fake_file = mock.mock_open(read_data=DUMMY_DATA)

        with mock.patch("__builtin__.open", fake_file ) as mock_file:
            assert get_users_from_file('somename') == ['matt']
```

`mock` is great when you can't avoid coupling between things that happen inside your code and the state of the rest of the universe.  However if you find yourself having to mock too much of your tests, it's a good indication that the code is overly coupled and needs to be refactored.

> Note: in Maya / python 2.7 we use `import mock` and `mock.patch('__builtins__.open')`.  In Python3 it's `import unittest.mock` and  `mock.patch('builtins.open')`

## Special Assertions

The basic `assert` method is the most common type of test but there are others as well.  The `TestCase` class also offers some special kinds of assertions that test other kinds of conditions. Using the special assert types is good testing style because it typically makes the point of the test more clear than manually managing the actual truth-tests.  Another nice advantage of the special asserts is that you you can provide a user-friendly error message which will be returned by the test runner in the event of a failure.

The entire list is [here](https://docs.python.org/2/library/unittest.html#test-cases) but here are some of the most useful ones:

#### assertSequenceEqual, assertItemsEqual, assertCountEqual

`assertSequenceEqual` is a convenient alternative to a bare assert when comparing sequences of different types (ie, you can use it to compare list [1,2,3] to tuple (1,2,3)).   `assertItemsEqual` ensures that two sequences contain the same elements without necessarily having the same order.  **NOTE: In Python3 this method `assertItemsEqual` was renamed (misleadingly) to `assertCountEqual`**


#### assertRaises

AssertRaises allows you to run some code and only pass the test if it raises the expected type of exception:

```
def test_divide_by_zero_raises(self):
    with self.assertRaises(ZeroDivisionError):
        mymodule.divide(0)
```

This is a very useful way to make sure that your code is providing useful errors for known fail cases.  Testing these is very valuable because it helps people write correct `try/catch` blocks around your functions.


#### assertAlmostEqual

This is handy for validating floating point values without having to worry about small changes introduced by, say, round-tripping a number from Python to JSON and back.   You can call it two ways:

```
self.assertAlmostEqual(a, b, places = 3)   # a is equal to be within 3 decimal places

self.assertAlmostEqual(a, b, delta = .01)  # a is within .01 of b
```


#### assertGreater, assertGreaterEqual, assertLessThan, assertLessThanEqual

These two are a useful shorthand for validating values without worrying about exact returns.