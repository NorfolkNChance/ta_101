Modules and packages are the fundamental building blocks of a Python codebase.  It's important to be deliberate about how they are organized.  

This page covers the key design questions you'll need to answer when spinning up a new unit of code, whether it's a single-file module, a multi-file package, or a complex nested package.  To avoid extra verbiage this page will use 'module' for all of these cases unless making an explicit point about things to do differently in a package vs a module.  Where this page uses the Python convention of [names](https://nedbatchelder.com/text/names.html), it  means both variables, functions, and classes.

# What goes into a module

A module is fundamentally a _namespace_: a collection of python names, which can be variables, functions or classes.  The first time the module is imported -- either in whole or in part -- any loose code inside the module will be run.  So for example a module like this:

```
import datetime

START_TIME = datetime.dateime.now()
print ("module initialized", START_TIME)

```

will always have the same value of `START_TIME` during a given program run.

Because of this one-time-only behavior, modules are Python's answer to singletons.  No matter how often, and from where, you call `import mymodule` you'll get back a copy of the `mymodule` namespace.  This makes modules a natural place for things like registries, loggers, and shared state.  

It's worth mentioning that modules are initialized from the top of the file down.  So this works:

```
def function_a():
	return function_b()

def function_b():
	return 999
```

but this does not:

```
CONSTANT = function_b()

def function_b():
	return 999
```

In the first example, `function_b()` cant' be called until the module is fully imported, so it exists when `function_a()` needs it.  In the second example, the line `CONSTANT = function_b()` will try to execute before `function_b()` is defined, creating an error at import time.

#### Keep module initialization code simple

It's a good idea to keep the functions which run at module import time as simple as possible.  A user of the code won't necessarily know what sequence of events is being triggered when importing a module.  If you've got a sequence of events which is potentially time consuming or failure prone you shoulud not make it part of a module initializer:

For example, this kind of registry is fine to put in to module code
```
REGISTRY = {}

def register_function (name, function):
	REGISTRY[name] = function


register_function ("print", print)
register_function ("sum", sum)
```

Whereas this is not:

```
import socket
_SOCKET = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
_SOCKET.connect(("www.python.org", 80))

```

Because it's possible that the `_SOCKET.connect()`  call could hang or except.  **Module initialization code should not create delays or risk raising exceptions**.  Risky and/or time-consuming operations should be explicit function calls.


# Basics

One of the most quoted lines in the zen of Python is

    Namespaces are one honking great idea — let’s do more of those!

If you're providing library or framework code -- that is, code you expect to be called by other code -- you can use Python's package style to help users navigate your offerings.

### Good naming

Good names are very important inside a package.  Large systems can contain dozens or hundreds of important names, and well chosen file names (which become namespace names) are important for organization.

For example, here's the package layoyut for  hierarchy for the module `planar`, which does 2-d geometry operations:

    planar
    |---- __init__.py
    |---- box.py
    |---- line.py
    |---- polygon.py
    |---- transform.py
    |---- util.py
    |---- vector.py

This gives a reader a pretty clear idea where to start looking for relevant material: `from planar.transform import Affine` makes intuitive sense to a reader.

### Nested vs Flat

A package author should think carefully about how names should be made visible to consumers.  A thoughtfully designed public interface to a module is important for usability and for uptake by other coders. Especially in a team situation like ours, gaining mindshare from your colleagues is an important measure of quality: it avoids duplication of effort and private knowledge.

There are two considerations to balance when designing the public face of a package:

#### Discoverability

Nested names are a valuable tool for readers -- but they go hand in hand with good naming practices.  As a coder trying to understand what you have to work with,  a package path like `animals.mammals.felines` is an obvious place to look for a `Cat` class.  `animals.datatypes.concrete` is less obvious.  300 lines down in a single file with animals, fish and birds is also obscure. 


#### Readability

On the other hand, as a consumer deeply nested names can also be a drag: having to write `animals.mammals.felines.some_cat_function()` many times over gets old fast.  In many cases the long chain of sub-packages is of interest only to the auther of the package and the user only wants `some_cat_function()` for the sake of readability.

When in doubt, you can leave this choice up to the user of the code: Python allows the importing code to facade long names at import time:

```  
    # these are all legitimate methods of shortening names

    # truncated imports
    import animals.mammals.felines as felines
    felines.some_cat_function()
    
    # cherry_picked names
    from animals.mammals.felines import some_cat_function
    some_cat_function()

    # aliases
	from animals.mammals.felines import some_cat_function as feline_func
	feline_func()

```

Since Python gives users the ability to change the feel of the imports, it's usually not necessary to artificially flatten a package hierarchy

### 'Hoisting'

Sometimes, however, you may need to move names around inside a package to resolve conflicts between the physical layout of the code and the contents of a namespace.  There are times when code wants to live in its own file for organization or source control reasons but the author wants to organize names into a more semantic sequence for users.  

In this case it's acceptable to use intra-package imports to move names into a the righ namespace. For example, the `__init__.py` of the  `planar` package does this:

    from planar.vector import Vec2, Vec2Array, Seq2
    from planar.transform import Affine
    from planar.line import Line, Ray, LineSegment
    from planar.box import BoundingBox
    from planar.polygon import Polygon

which allows users to do `from Planar import Vec2` and the like (the actual implementation also uses an `__all__` statement to explicitly list the names that the module will export: there's more about `__all__` down below.) 

As a style choice, hoisting makes sense when the primary reason for breaking up the package was for developer convenience rather than semantics.  It allows you to provide a public face that is curated without forcing your internal file layout to exactly match the public face. 

If you do provide a hoisted public interface, it's a good idea to include a module level comment to make it clear what the expected style of import should be.  You may want to use underscored file names (see below) to discourage direct deep imports if you want to control how users import the package. 

### Cross imports

Often you'll need to import code from within the same package. 

In order to be ready for Python 3, you have to use [explicit imports](https://www.python.org/dev/peps/pep-0328/) for intra-package imports.  It's a good idea to include `from __future__ import absolute_import` at the top of Python 2.7 files -- that will make sure that the behavior is the same as Python 3.

It's **very important to keep makes sure that you don't create circular dependencies inside a package**:  if you've got sibling files inside your package you should be clear about which ones of them are "producers" and which are "consumers" of names: a file should not be both. 

So this example, where `a.py` is used by `b.py` and `b.py` is used by  `c.py` is good:

```
    # acceptable sharing

    # in package.a.py
    def func_a():
    	pass

    # in package.b.py
    from package.a import func_a

    def func_b(): 
        return func_a() + 1

    # in package.c.py
    from package.b import func_b
    def func_c():
    	return func_b() * 10
```

However **this is dangerous**

```
    # Circular dependendies.  Don't do this!

    # in package.a.py
    from package.c import func_c  # <- this creates a circular dependency
    def func_a():
    	pass

    # in package.b.py
    from package.a import func_a

    def func_b(): 
        return func_a() + 1

    # in package.c.py
    from package.b import func_b
    def func_c():
    	return func_b() * 10

```

It's common to isolate the 'producer' modules with abstract names like `core` or `lib` or `shared`.  **'Producer' modules should not import each other!**


# Designing a public face

A module should be as explicit as possible about what names it intends for consumers to use; it's good to let readers know what you are expecting them to take away and what not.  This is important for helping us understand the scope of potetntial changes -- if somebody is using one of your functions in a way you did not anticipate, a change which seems innocent to you can cause bugs elsewhere.

On the other hand, Python does not have compile-time or language level privacy guarantees -- it relies on a "consenting adults" model of security.  It also offers competing conventions for indicating your intentions, which can lead to confusion when picking up other people's code.  The goal of this doc is to provide some guidance on what the various conventions do and don't do.


## Start with tests
Tests are the strongest guarantee of a public interface in Python, because they are the only ones that can't be ignored by a consumer.  

Say you have a module `foo` and you intend the public interface of `foo` to be two methods and a class:


```
import random

class FooStruct (object):

	def __init__(self, name, value):
		self.name = name
		self.value = value

	def func(self):
		print "my name is ", self.name, "and I am ", self.value

REGISTRY = 0

def bar (name):
	next_up = _dont_call_me_from_outside()
	return FooStruct(name, next_up)


def baz (name):
	value = random.RandInt(100)
	return FooStruct(name, value)


def _dont_call_me_from_outside():
	global REGISTRY
	REGISTRY += 1
	return REGISTRY

```

If you add these tests around the module you can be sure that it always retains this interface (or that it does not get into the build if it fails to retain the interface):

```
from unittest import TestCase
import foo

class TestFoo(TestCase):

	def test_bar(self):
		result = foo.bar('name')
		assert isinstance(result, foo.FooStruct)


	def test_baz(self):
		result = foo.baz('name')
		assert isinstance(result, foo.FooStruct)


	def test_FooStruct_constructor(self):
		result = foo.FooStruct('name', 1)
		assert result is not None


    # this is another way to guarantee the presence of a function
    def test_callable(self):
    	assert callable(foo.bar)
    
    # this is another way to guarantee the existence of a class
    def test_FooStruct_exists(self):
    	assert foo.FooStruct

```

These tests don't actually prove the code works: they just verify that the code provides some what you expect. Even at this level that's a valuable service: they guarantee that the signatures and return types of the methods and the class are the same as they have always been.  If you inadvertently changed the `bar()` method to take two arguments, the first test would fail -- alerting you that something was wrong before the code was used.   **Because tests actually guarantee some things about behavior, they are the strongest guardians of a public interface.** You can always get around other privacy methods, but you can't fool the tests.

#### When to use?
When you're getting a module ready for public consumption,  interface tests are a key way to set the interface in stone

#### When not to use?
Interface tests are boiler-plate-y. You want to add them for major releases. 

## Underscore names

There's a strong convention im Python that a leading underscore on any name indicates "I am private, don't mess with me".  This still _just_ a convention; it does not have any weight at runtime, and a caller can always decide to ignore it.  However it's a strong signal of authorial intent.

By default, names beginning with underscores are not included in star imports:

```

# module foo.py

def public():
	return _private()

def _private():
	return "magic"

```

`from foo import *`  will only yield the name `public`.   However there's no way to stop a caller from using `from foo import _private`.  

Underscore methods are a good habit to encourage because they are relatively low maintenance: the 'private' state of the method is clearly signalled.

Python also supports names with two underscores ('dunder' names).  These are really only intended for use in class variables, they are slightly mangled at compile time to prevent name collisions among derived classes.  Unless you're working on a complex class hierarchy where you don't your inheritance tree in advance (ie, you're expecting other coders to subclass your classes and want to be sure there are no accidental name clashes in the family) you don't need dunder names.  They don't otherwise enhance 'privacy'.  For more see [here](https://stackoverflow.com/questions/1301346/what-is-the-meaning-of-a-single-and-a-double-underscore-before-an-object-name)


#### When to use
Inside a single file module, underscores are the best way to indicate your intention to make something local.  Use them liberally. The great advantage of using underscore names is that anybody who decides to 'cheat' will be fully warned of the danger of using underscore names

#### When not to use

Leading underscore names are ugly. If your file is full of underscore names and you don't like the way it reads, you may want to use inlining (see below)

Only use double-underscore names for class methods or variables.

##  Private modules

Python does not have a concept of 'private' modules or packages.  However python sub-package names can begin with underscores, just like other names can.  If an entire file is really a deep-down implementation detail that you really don't want users to touch, you can use the leading underscore name to telegraph that fact. 

##### When to use

When you have a lot of implementation specific stuff in one place and you want to broadcast "don't touch these details".  If you use an underscored module it's a good idea to structure the so it retains the underscored module name so the 'private' meaning is not obscured:

```
    # note the relative imports here... 
	from . import _helpers 

	def choose_class(selector):
		result_class = _helpers.find_class(selector)
		if result_class:
			return result_class()
		else:
			return _helpers.get_fallback_class()	
```

##### When not to use

Underscored module names only make sense when you want to 'privatize' a large chunk of code in one go.  Most of the time you should use more fine-grained access control mechanisms.  

If you find yourself tempted to import an underscored part of a package from outside, that's a bad sign:

``` 
	# don't do this
	from animals._helpers import some_secret_func()
```


## Inlining

If you're very worrired about keeping sensitive methods or classes from being misused, you can **inline them:** nest them inside another function or class.  This keeps them out of the namespace of the module and gives readers a strong hint that the items are intended to be internal use only.  

Inlining also lets you make use of [closures](https://medium.com/@dannymcwaves/a-python-tutorial-to-understanding-scopes-and-closures-c6a3d3ba0937) which are can be more performant than conventional function calls.

Here's an example of using inlining to hide implementation details:

```
   # conventional helper function with underscores   
   def _make_a_number():
       return 99

   def _format_number(num)
  		return "NUMBER: {}".format(num)

   def main_func():
       seed_value = _make_a_number()
       return _format_number(seed_value)


  # inlined helper functions

   def main_func():

       def make_a_number():
           return 99

       def format_number(num):
           return "NUMBER: {}".format(num)

       seed_value = make_a_number()
       return format_number(seed_value)
```

For functions, this is actually the most aggressive form of privacy because the inlined helper functions are only accessible through very fancy tricks.  

#### When to use

If a function or class will benefit from using closures, or if you want to avoid the aesthetics of lots of underscores in your module namespace.


#### When not to use

The primary disadvantage of inlined functions is readability:  Too much deep nesting renders a file very hard to read.

Be very careful about deeply nested functions, they quickly become unreadable.


## `__all__`

A module can also defines an iterable called `__all__` which is allows you to explicitly indicate which names in the module are expected to be public.   If you provide and `__all__`, then `from foo import *` will only return names that appear in the `__all__` variable.  

The primary use of `__all__` is to explictly control what users will get if they do a star import on your code. For example :

```

__all__ = ['foo', 'bar']

def foo():
    return 1

def bar():
    return 2

def hidden():
    return 3

```

Writen this way only `foo()` and `bar()` will be imported with a star import. However there is nothing stopping a user from calling `from foo import hidden`.   
Like underscore names, `__all__` is good for hiding implementation details. However it's more fragile than underscores: It's up to an author to maintain the link between function or class names and entries in the `__all__` list by hand.  A typo in the `__all__` will not allow the file to be imported.  Renaming a method or a class has to be mirrored correctly in the `__all__`.  New names   

#### When to use

The primary use case for `__all__` is in modules that are explicitly intended to be star-imported. Typically these will be modules that expose a lot of 'legos' for users:  for example, it makes a lot of sense to star import `PySide2.QtWidgets` because the alternative might be dozens of individual imports.  

If you're using `__all__` for this type of application, the module docstring should clearly say "this module is designed for star imports" so that readers know you've thought it through.

Another use for  `__all__` is to avoid the visual clutter of underscores in a module with a very high implementation detail to public api ratio.  For example, you might have a module that does a very complex multistep operation in Maya, with dozens of individual functions that handle smaller parts of the operation but which don't make any sense in other contexts.  Or, you might have a module with a complex initialization routine that runs in line and creates a lot of state you want to keep hidden.  If you've got dozens of private names and only a few public ones, `__all__` can help keep things readable.  


#### When not to use 

The downside of `__all__` is that it's not contextual.  You have to accept the risk that a reader who skips the all might look at your file an see an appetizing method without bothering to check the `__all__` for permisssion.  

If you are going to rely on `__all__` it's a good idea to reinforce that choice by keeping the public members together near the top of the file along with a comment indicating where the line between 'public' and 'private' lies.

