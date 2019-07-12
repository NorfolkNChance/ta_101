Modules and packages are the fundamental building blocks of a Python codebase.  It's important to be deliberate about how they are organized.  

This page covers the key design questions you'll need to answer when spinning up a new unit of code, whether it's a single-file module, a multi-file package, or a complex nested package.  To avoid extra verbiage this page will use 'module' for all of these cases unless making an explicit point about things to do differently in a package vs a module.  Where this page uses the Python convention of [names](https://nedbatchelder.com/text/names.html), it  means both variables, functions, and classes.

# What goes into a module

A module is a lot of things, but fundamentally it's a _namespace_: a collection of python names, which can be variables, functions or classes.  

Because a module is a namespace, its important to think about how to group work into meaningful units.   Good namespaces are clear, unambiguous, and scoped tightly.  They should be hierarchical: a top level name should be a good guide to what's available at lower levels.  So, a module named `animals` could logically have children like `animals.mammals` or `animals.flying`  but it probably does not want a child like `animals.aircraft`

## Module-level code

Most of what we use modules for is to group functions and classes. 

However, modules can also run code at import time -- a very useful feature, but one that has to be used carefully.  In some ways a module is like a class and the module-level code is like the class initializer.

The first time the module is imported -- either in whole or in part -- any loose code inside the module will be run.  So. for example, a module like this:

```
# timekeeper.py (or the __init__.py of a 'timekeeper' package)

import datetime

START_TIME = datetime.dateime.now()
print ("module initialized", START_TIME)

def session_start():
	return START_TIME

def elapsed():
	return datetime.dateime.now() - START_TIME

```

will always have the same value of `START_TIME` during a given program run.  All calls to `elapsed()`  will return the difference between calling time and the time the module was first imported.

Because of this one-time-only behavior, modules are Python's answer to singletons.  No matter how often, and from where, you call `import mymodule` you'll get back a copy of the `mymodule` namespace.  This makes modules a natural place for things like registries, loggers, and shared state. 

Conversely, [you almost never want to write a 'singleton class' in Python.](http://www.informit.com/articles/article.aspx?p=2131418&seqNum=5)

## `if __name__ == '__main__':`

This is a common python pattern which helps you distinguish between code which always runs at module import time and code which _only_ runs if the python file is being executed.  It's got two primary uses:

* **Script modules**  If you want a module to do double-duty as a code unit that other code might use but also as an executable script, use `if __name__ == '__main__'` to isolate the code only runs when the module is being a script.
*  **Testing** Often when you're working on a module, it's helpful to throw an `if __name__ == '__main__'`  block into the file so you can repeatedly run it from a commandline as you evolve tests.

This is a handy workflow when you're in development.  However, it's bad practice to publish code with a test-only block inside it: if the tests are still valuable, convert them to proper unit tests, and if they aren't needed delete them. The only time you want to publish an `if __name__ == '__main__'`  block is when your module is intended to be run interactively at least some of the time.

## Keep module initialization code simple

It's a good idea to keep the functions which run at module import time as simple as possible.  A user of the code won't necessarily know what sequence of events is being triggered when importing a module.  If you've got a sequence of events which is potentially time consuming or failure prone you shoulud not make it part of a module initializer:

For example, this kind of registry is fine to put in to module code
```
REGISTRY = {}

def register_function (name, function):
	REGISTRY[name] = function


register_function ("print", print)
register_function ("sum", sum)
```

Whereas this is **not**:

```
import socket
_SOCKET = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
_SOCKET.connect(("www.python.org", 80))

```

Because it's possible that the `_SOCKET.connect()`  call could hang or except.  **Module initialization code should not create delays or risk raising exceptions**.  Risky and/or time-consuming operations should be explicit function calls.

## Execution order 

Modules initialization code is run from the top of the file down.  So this works:

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

The entire module is evaluated the first time somebody tries to import anything from the module.  That means even of you do 

```
from mymodule import just_one_thing
```

all of the module initialization code will run at the time the import statement is hit.  That's another reason to keep module initializers short and sweet.

---

