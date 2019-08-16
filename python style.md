Python Style
=================

First and foremost, think about **how Python wants to be written**.  Simplicity and readability are key values in python code.  There's a reason a lot of core Python developers use the word 'beautiful' a lot when talking about programming.  Making good use of common Python idioms will make your code easier to read and maintain for everybody on the team.

## imports

### One import per line

    import module
    import othermodule

NOT 

    import module, othermodule

### Group imports logically

> see doc

### One Dot
Where possible, aim for 'single dot' imports, so the reader knows where code came from but does not have to parse long lines of identifiers:

    import module
    import package.subpackage as subpackage

    module.do_something()
    subpackage.do_something()

This is a compromise between clarity -- this name came from some other file -- and preserving line length (ie, no package.subpackage.subsubpackage.function())

The exception is when you are explicitly moving names around for API management.  If a package defines a number of classes in several files, but wants to present it's api as a single namespace, the package can import its own 'children' by name. 


### Within a package

Inside a package, prefer absolute imports.  For a package like this:

     pkg
     +---- __init__.py
     +---- core.py
     +---- mod1.py
     +---- mod2.py

if `mod1` wants to import shared it should be

    import pkg.mod1 as mod1

### Avoid from ... import ....

Avoid importing names directly with `from`:

    # DON'T
    from somemodule import somename

This syntax makes reloading a module unreliable.   There are two tactical exceptions to this rule:

1. Very performance intensive code, which benefits from a flatter search space
2. Code which makes heavy use of an API where the full paths are redundant; for example a file whose only job is to define a complex QT UI with lots of widgets.

### Avoid dynamic imports

Dynamic imports should be avoided if at all possible; they are not to be used casually.  They make it too hard to reason about what's going on at runtime.

In the (rare) cases where they are needed, use `importlib` rather than `imp`

If you need a switched import based on context, use a conditional rather than a dynamic import.  This is still less than ideal, but will survive refactors.

    if project == projects.agate:
       import menus.agate as tools
    elif project === projects.class4:
       import tools.class4 as tools
    else:
        raise RuntimeError ("Unknown project")


### Avoid star imports

Star imports should be avoided under almost all circumstances; they are impossible to reason about and can create bugs if there are name clashes. 

### Dont import inside functions

Imports should be static and happen at the top of the file.  The only exceptions are:

1. functions whose whole job is to perform dynamic imports (say, based on user input from a UI)
2. imports which have high initialization costs (eg, pymel) and which might not ever be used


## modules

### Keep file sizes manageable

If a module is larger than 500 lines, it's time to think about refactoring into a package with smaller pieces. 

> If you're breaking up an overly large file, make sure you have tests which guarantee the public names in the existing file before moving the pieces around.  Then turn the module into a package and import the names from the sub-packages into the old namespace to preserve old code.

### keep __init__.py clean

The only things that should go into an __init__.py are:

1. Package level comments
2. hoisted imports, which should be explicit rather than stars



### Initialization

1. initialize constants and module state at the top of a file
2. if you have to run code (as opposed to just assignements) run that at the bottom of the file so it runs after all the other names in the file are defined

    imports
    CONSTANTS
    module_vars
    defs and classes
    initialization logic


In larger packages designate a single subpackage as the shared initializer (typically you'll want to call that `shared` or `core`).   Any sibling subpackages which need the initialization should import that subpackage.  Using module-level code rather than an explicit initialize function automatically prevents repeat initalizers without extra work.

Don't include disk access, network calls, or heavy duty computation in initialization code; users should have to opt in to the potential speed hit.


## The toolbox

<a id='modules'></a>
### Modules

Modules are a vital tool for keeping python code organized.  Keeping related code together in a well-named module allows for a good mix of descriptive names wihout redundancy.  Good modules reduce the need for extremely long function names, and allow for more flexibility in layout

    import texture_tools
    texture_tools.get_uvs_fom_selected()

and 

    import uvs
    uvs.from_selected()

Don't write a class just to create a holder for methods  -- make a nested python module instead. 

Python modules are also the right way to handle shared state which in other languages requires a singleton.  For example, you might want to have project related code for managing data about your working environment.  You would not want different parts of the same application to disagree about that kind of data.  In Python delegate that code to a module instead of trying to write a singleton class:  Modules are globally accessible, only run their own code once, and they are 'singletons' by default.  

In general, "singleton" style shared state is something to avoid whenever possible in any case.  Too much shared state makes it too easy for bugs to crop up in untraceable ways.  However when you do need to share information, use the module mechanism as the easy, Pythonic way to maintain shared data.

<a id="basics"></a>
### Lists, Tuples and Dictionaries -- know the basics

Collections are the heart of Python programming -- they're super useful and reduce a ton of the boring boilerplate common in other languages.

It's important to get good use out of them, which means a couple of basic things:

#### Use idiomatic looping

Use  

    for item in my_list:
       do_thing(item)

for most loops and 

    for index, value in enumerate(my_list):
       do_thing(index, item)

if you need the indices.  Don't use `for i in range(len(my_list)))`

For dictionaries, prefer 

    for item in my_dict

for most things, and
   
    for item in my_dict.keys()

if you might be adding or removing entries in the loop.

#### Use 'in' for content checks:

Use 
   
    if "x" in my_list

and

    if "x" in my_dict

and

    if "x" in my_tuple


#### Learn slicing

One of the best tools for writing compact, readable Python is [slice notation](https://medium.com/@adamshort/python-slicing-72f76bb36e31).  It's very useful for a wide variety of tasks -- anything from reversing a list to subsetting it to taking every Nth item can be done concisely with slices.

An extra incentive to learn slicing is [`itertools.islice`](https://docs.python.org/2/library/itertools.html#itertools.islice) which offers the same functionality but as an iterator -- it's an elegant way to express an idea like "take every third item from this list, starting at number 10000 and counting down"  without the expense of creating a new in-mempory copy of the list.  See [Use Generators](#generators), below.

#### List comprehension are great -- if comprehensible

Most of the time `my_list = [x for x in range(100) if x %2 ==0]`  is better than a for loop.  However if you're finding it hard to fit a comprehension onto a single like it's probably too complex and ought to revert to being a loop.


#### Prefer tuples to lists if you can

Tuples are slightly faster than lists, and they have less overhead if you don't expect their contents to change.  Whenever you're simply answering a question like "what are the objects in this Maya scene using this shader" -- it's better to return a tuple instead of a list, since it's the correct answer now and user's can't accidentally change it.  There's no way to, say, try to change a tuple while loopong over it.

Tuples are also faster to create as literals.  List initializers are fast:

    data = ['a', 'b', 'c']

but tuple literals are faster (note, btw that it's the _commas_, not the parens, which define a tuple) 

    tuple = 'a', 'b', 'c'

So tuples are a natural fit for constants lookups or other data you don't expect to change at runtime.

Tuples can also be used as keys in a dictionary, where lists cannot.  So you could for example represent something like a chessboard as dictionary using tuples as keys:

    Chessboard = {(0,0): 'Rook', (0,1), 'Pawn', ... }

This is much more efficient than having an actual nested 2-d array to represent the board.

#### Prefer namedtuples to tuples for structured data

[`collections.namedtuple`](https://pymotw.com/2/collections/namedtuple.html)offers an excellent alternative to dictionaries and custom classes for structured data.  It's far better to write

    if person.age > 21

than 

     if person[7] > 21

 namedtuples make for more readable code with very little work.  They are also cheaper than dictionaries and -- being immutable, like tuples -- they are less prone to accidental mutations.

#### Dictionary idioms

Dictionaries are one of Python's best features -- an excellent way to handle information flexibly. Using them idiomatically is a key to good Python style:

* Don't forget about dictionary comprehensions:  {k: v  for k, v in zip(keys, values)}
* Check for inclusion with `if value in my_dict`.
* Loop over the keys and vales together with `for key, value in my_dict.items()` (or `.iteritems()` to save memory).
* Use `my_dict.get(key, fallback_value)` rather than checking to see if a key already exists.  Use `my_dict.setdefault(key, value)` to do the same thing _and_ to ensure that the key is present in future.
* Remove dictionary keys and values with `del my_dict[key]`, but get-and-remove in one operation with `my_dict.pop(key)`  using `del` makes it clear that you intend to remove a key
* Use [`collections.defaultdict`](https://docs.python.org/2/library/collections.html#collections.defaultdict) if you have to do a lot of checking to see if a dictionary already has what you need instead of hand-writing an if-check.  

Dictionaries make a very useful alternative to long string of `if - elif - else` comparisons, particularly if you're routing to different code based on some value.  For example if you were tring to choose between  handler functions based on a selector string:

     options = {
        'a': a_handler,
        'b': b_handler,
        'c': c_handler
     }
     func = options.get(selector, fallback_handler)
     func()

is more compact and easier to extend than

    if selector == 'a':
        a_handler()
    elif selector == 'b':
        b_handler()
    elif selector == 'c':
        c_handler()
    else:
        fallback_handler()

<a id="generators"></a>
### Use Generators

One of the most important things you can do to improve the speed of your Python is use generators and iterators instead of creating large in memory lists.

These two pieces of code produce the same result:
    
    test = 0
    for n in range( int(40e6)):
        test += n

and 

    test = 0
    for n in xrange (int (40e6)):
        test += n

however the second one is **40% faster** because it does not create a complete list in memory of 40,000,000 integers -- it just processes the numbers one at a time.

Whenever possible, use the [`yield`](https://jeffknupp.com/blog/2013/04/07/improve-your-python-yield-and-generators-explained/) keyword to have your functions generate results that can be iterated over, rather than lists which have to be created in memory.  If the caller needs the entire list, it's easy to turn a generator into a list or a tuple.  But if the caller does not need the whole thing in one go -- if they just want to process items one at a time -- widespread use of generators makes for faster, more memory efficient code.  

It's also easy to [chain generators together to create fast pipeline-style code](https://brett.is/writing/about/generator-pipelines-in-python/) which does filtering or transformations in small, composable pieces.  The helps with both reusability and performance.

<a id="tryblocks"></a>
### Try-Except-Finally

`try` and `except` are basic parts of Python.  Their lesser-known sibling `finally` however is a very powerful tool for writing cleaner code.  The basic structure is :

    try:
        do_something()
    except:
        handle_problems()
    finally:
        clean_up_after_yourself()

In a `try/except/finally` block the code in the `finally` section is guaranteed to execute whether or not the `try` block raised an exception.  This is important because it allows you to handle cleanup tasks knowing that they will execute regardless of whether or not the code above them succeeded or failed.  Consider something like this:

    try:
       output = open('filename', 'wt')
       output.write( 99 / x)
       output.close()
    except IOError:
        print "could not open file"

This looks OK -- the code correctly handles a case where, say, 'filename' is write-locked.  However if the variable `x` is zero, the code will fail with an uncaught `ZeroDivisionError` and the file handle will be left open, which will leave an undeletable file on disk until the python interpreter is forcibly closed.  A `finally` block will make sure that the cleanup happens no matter what:

    try:
        output = open('filename', 'wt')
        output.write( 99 / x)
    except IOError:
        print "could not open file"
    finally:
        output.close()

In this version, `output` will be properly closed no matter what else happens.  In Maya programming this is often a vital tool for making sure that the scene is restored to a legitimate state if one of your tool operations goes awry.

<a id="contexts"></a>
### Context managers

A variant on the same theme is the context manager: the Python construct that start with `with`.  A context manager is an excellent way package up work that has a defined beginning, middle and end in a readable, but also safe way.

For example, Maya wants you to do this to create and work in a new namespace:

     cmds.namespace(set = ":")  
     #  you have to go back to the root namespace, or you'll create a child
     cmds.namespace(add = 'new_ns')
     cmds.namespace(set = 'new_ns')
     
     # ... do work here ...
     cmds.namespace(set = ":")  

     # return to root, or other work will be done in your namespace 

This is all very simple, and maya vets don't think it's a big deal.  However, for four lines of boilerplate it has four significant weaknesses:

1. If you forget to reset the namespace at the top, the rest of the code will quietly produce data in the wrong place
2. If you mistype the namespace in the second or third lines, the code will fail. Because the typo is in a string, you won't know it until runtime.
3. If you forget the reset the namespace at the end of the block, you'll be changing the behavior of all the code that runs after this.
4. If the work code raises an exception, the namespace won't be reset and all future code will also fail.

Luckily Python has a great native mechanism for dealing with this kind of setup->work->cleanup paradigm: the `with` statement.  It's not a lot of work to create a context manager that handles the boilerplate above:

```
import maya.cmds as cmds
class Namespace(str):
    def __init__(self, name):
        self._name = name
        self.last_namespace = None
        self.namespace = None

    def __enter__(self):
        self.last_namespace = ":" + cmds.namespaceInfo(cur=True, fn=True)
        
        if self._name.startswith(":"):
            should_add = not cmds.namespace(exists = self._name)
        else:        
            should_add = self._name not in (cmds.namespaceInfo(lon=True, sn=True) or [])
            
        if should_add:
            cmds.namespace(add = self._name)
        cmds.namespace(set = self._name)
        self.namespace =  ":" + cmds.namespaceInfo(cur=True, fn = True)
        return self

    def __exit__(self, *exception):
        cmds.namespace(set = self.last_namespace)
        return False  
        
    def __str__(self):
        return self.namespace or 'invalid-namespace' 

    def __repr__(self):
        return 'Namespace("' + self.namespace  + '")' or 'Namespace(" + self.name + ")'
```


That's a quite a few more lines to write -- but once it's written all other namespace jobs can handled much more cleanly:

    with Namespace('my_namespace'):
        # do work in 'my_namespace',
        # and return to whatever namespace you were in
        # when done

Formally you could do the same work in `try-except-finally` structure and be guaranteed to get the same results.  From a readability standpoint, however, the `Namespace` context manager is a big win -- it takes a series of steps that are all related, all necessary, and all pretty common in Maya programming and renders them both safe and easily readable at the same time.

You may note that `with` blocks have a generic similarity to `try-except-finally` constructs.  Generally the former are best for things you have to do a lot, like namespaces and the latter are the fallback for one-offs.


<a id="closures"></a>
### Closures

Python [closures](https://www.programiz.com/python-programming/closure) are a very important tool for sharing data without the need for elaborate class structures.  The rules have some interesting subtleties, but basically they boil down to this:

1. When you create a new scope -- a function or a class -- it automatically inherits the names defined in its current scope. If you declare a variable first and then declare a function right afterwards, that function gets access to that variable:
```
    test = "hello world"
    def do_test():
        print test
    # hello world
    print test   # here we're in the outer scope again
    # hello world 
```
2. Names inherited via a closure can be _read_ (as in the above example), or _mutated_:
```
    test = {'hello': 'world'}
    def do_test():
        test['hello'] = 'sailor'
        print test
    # hello sailor
    print test  # back to the outer scope -- the change sticks
    # {'hello' : 'sailor'}
```
3. However, inherited names _cannot_ be re-assigned.  So, the obvious extension of the first example will not work as expected:
```
    test = "hello world"
    def do_test():
        test = 'hello sailor'
        print test
    # hello sailor
    print test # back to the outer scope: no change!
    # hello world
```

This key difference here that assignment (that is, the `=` operator) does not work its way back up the outer reference chain; instead it creates a new variable that masks the inherited version inside the local scope only. 

The inheritance rule can be a bit intimidating, but closures are a very powerful tool for simplifying Python code particularly in a maya context.  Probably the best example is creating GUI, which often requires different widgets to know about each other.  Here's an example of a simple GUI that uses closures to let the different widgets communicate without the need for an elaborate class structure:

```
    def closure_window():
        window = cmds.window(title ='closure example')
        layout = cmds.columnLayout(adj=True)
        rowLayout = cmds.rowLayout(nc=3)
        increment = cmds.button(label = 'plus')
        decrement = cmds.button(label = 'minus')
        counter = cmds.intField()

        def get_counter():
            return cmds.intField(counter, q=True, v=True)

        def set_counter (val):
            cmds.intField(counter, e=True, v=val)

        def add(_):
            set_counter(get_counter() + 1)

        def sub(_):
            set_counter(get_counter() - 1)

        cmds.button(increment, e=True, c=add)
        cmds.button(decrement, e=True, c=sub)

        cmds.showWindow(window)
        return window

```

Here, the closure allows the increment and decrement buttons to know which field to affect even after the function has fired and the window has been created.  You could achieve the same effect with a class, using instance fields and instance methods to connect the different widgets together. In complicated cases that's still a good way to go. However closures provide many of the same benefits with much less overhead and should be part of your design process.  

Closures are particularly attractive because they are space efficient -- however you do need to avoid going too deep, or your readers may not be able to figure out where your variable names are coming from. If you are inheriting a variable several screens away from where it originates, at least leave a comment indicating where the name comes from. The `ALLCAPS` convention for module-level constants is also a good way to remind readers when they may be seeing a closure variable.



### Decorators

Python decorators are a powerful tool for making simpler code. A decorator is basically just a way of wrapping a ordinary function to add some additional functionality.  For a toy example:

```
    def rightslash(original_function):
        def right_slashed(*args):
            result = original_function(*)
            return result.replace('\\', '/')

        return right_slashed

    @rightslash
    def working_directory():
        return os.getcwd()

    @rightslash
    def parent_directory():
        return os.path.dirname(__file__)

```

This would allow you to take a bunch of file management functions and ensure they all returned right-slashed pathnames instead of left-slashed ones on windows.  

You could of course do the same thing by doing the replace operation in every one of the functions, but the decorator version has two key advantages. First, it *spells out its own intentions clearly* -- it promises the reader that a function will (in this example) return a certain kind of data. Second, it's *easily extensible*.  Say you discovered that you did not want to right-slash paths if they were UNC-style paths like `\\tajiri\team\steve` -- going back and adding the same logic to a dozen different file handling functions would be tedious and an invitation to bugs, while fixing a single decorator function would be far quicker, easier and safer. 

Decorators are a very powerful tool for enforcing consistency across functions which are otherwise independent of each other.  For example, a Maya library that dealt with geometry could use a decorator to make sure that it's functions transparently found the shapes associated with transform arguments in a robust and reliable way instead of forcing dozens of functions to use similar but not-quite-identical ways to find the geometry.  Or, a file exporter could use decorators to make it easy to provide common logging and error handling for the many individual operations that make up an export. Whenever you're faced with the problem of coordinating similar behavior across a large range of functions (or even of classes -- classes can be decorated too) decorators are an excellent tool, offering the standardization that other languages get from class hierarchies without the accompanying complexity.

The one thing to remember about decorators is that they shouldn't interfere with readability:  a decorator that automatically right-slashes file names won't surprise readers, but one which causes a function that looks like it returns string to return numbers instead will generate a lot of confusion.

## Afterword: The proper uses of magic

Python has a high magic quotient; you can change a lot about the behavior of basic Python entities, allowing you to customize many aspects of how your code looks and acts.  In the right circumstances this is a very powerful tool: you can write code that more clearly and concisely expresses its intent if you know how to use advanced techniques.  However, this is a power that should be used _sparingly_: it's very easy for a system to become so sophisticated that  it's incomprehensible to any one except the author -- or, sometimes, _including_ the author once a few months have passed.

There's no hard and fast rule to knowing when the time for magic has come, but a good rule of thumb to start with is actually aesthetic.  Consider the case of a class that acts as a vector for vector math:

```
class Vector (object):
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

    def add_in_place(self, othervector):
        self.x += othervector.x
        self.y += othervector.y
        self.z += othervector.z

    @classmethod
    def add (cls, vector1, vector2):
        return Vector (vector1.x + vector2.x, vector1.y + vector2.y, vector1.z + vector2.z)

    # imagine similar methods for subtraction, multiplication and so on...

v1 = Vector (1, 2, 3)
v1.add_in_place(Vector(4,5,6))
# Vector(5,7,9)

v2 = Vector.add(v1, Vector (4, 2, 0))
# Vector(9,9,9)

```
        
This works pretty well -- but it violates a basic expectation for vector math, namely, that you can use standard mathematical notations (with all the other things they imply, like order of operations, commutation, and so on). Here a small investment in 'magic' makes sense:

```
class Vector (object):
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

    # inplace addition
    def __iadd__(self, othervector):
        self.x += othervector.x
        self.y += othervector.y
        self.z += othervector.z

    # addition
    def __add__ (self,  vector2):
        return Vector (self.x + vector2.x, self.y + vector2.y, self.z + vector2.z)

    # there are similar magic methods for subraction, multiplication etc.

v1 = Vector (1, 2, 3)
v1 += Vector(4, 5, 6)
v2 = v1 + Vector(4, 2, 0) 
```

The functional part of the code is no different but this class operates in an appropriate problem space.  The win in readability is strong, but the underlying relationships are still easy to parse.  That's some white magic.

On the other hand there are times when the magic is too deep and you risk tangling with Things Better Left Alone. Python [metaclasses](https://jakevdp.github.io/blog/2012/12/01/a-primer-on-python-metaclasses/) earned this quote for a reason:
> “Metaclasses are deeper magic than 99% of users should ever worry about. If you wonder whether you need them, you don’t (the people who actually need them know with certainty that they need them, and don’t need an explanation about why).” — Tim Peters

There really are legit applications for metaclasses, but they are rare (and often, the temptation to find uses coincides a bit too much with learning about metaclasses for the first time).  If you're considering a metaclass -- or other highly involved techniques like runtime [monkey-patching](https://www.youtube.com/watch?v=ZpJxwpyJpq4) -- be sure you're actually solving the problem and not embracing cleverness for its own sake.
