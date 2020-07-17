---
layout: post
title:  "Python memory management in Jupyter Notebook"
author: mikulskibartosz
tags: [ 'Python', 'Performance Engineering' ]
description: "How to avoid memory leaks in Jupyter Notebook"
featured: false
redirect_to: https://mikulskibartosz.name/python-memory-management-in-jupyter-notebook
hidden: false
excerpt: "How to avoid memory leaks in Jupyter Notebook"
---

In this article, I am going to show you how memory management works in Python, and how it affects your code running in Jupyter Notebook.

First, I have to describe the **garbage collection** mechanism. A garbage collector is a module responsible for automated allocation and deallocation of memory. The memory deallocation mechanism relies on two implementations: **reference counting and generational garbage collection**.

# Reference counting

Every time you create a new value (whatever it is, a number, string, object, etc.) Python needs to allocate memory for that new value. When it happens, the garbage collector not only gives you the memory you need to store the value, but it also creates a counter. The counter is used to count references to the value.

Every time you assign the value to another variable or a property of an object, pass it as a parameter or add the value to a collection, the counter gets increased.

Similarly, when you remove the value from a collection, the program exits a function (so it cannot use the parameter anymore), you set the value to None, override it with a different value or remove the variable (using the "del" statement), the counter gets decreased.

**When the counter reaches zero, the garbage collector deallocates the memory occupied by the value**, because the value is no longer used. Of course, when the value gets removed, the variables it referenced will get their reference counts decreased, and as a consequence, more memory may be deallocated.

The reference counting implementation is fast and easy, but it is too simple to deal with cyclic references. A cyclic reference happens when an object references itself, or a few objects reference each other in such a way that object A references the object B. Object B references object C, and object C references object A again. 

In such situations, the objects the reference counting will never deallocate the memory used by those objects even if no variables are pointing to any of them. Because of that generational garbage collector was added to Python.

{% include info.html %}

# Generational garbage collector

## Memory deallocation

To avoid problems with cyclic references, the generational garbage collector checks whether the object is **reachable from the application code**.

If there is a variable pointing at the object, it is reachable. If a reachable object has a reference to another object, the second object also becomes reachable.

The garbage collector can safely remove objects which are not reachable from the application code. As you see, the process of checking whether an object is reachable is not as straightforward as counting references, so it is not as fast as deallocating memory when reference count drops to zero.

Because of that, **Python does not run the generational garbage collector every time a reference is removed**. Such a garbage collection depends on the concept called object generations and configurable thresholds. Let's take a look at the details.

## Object generations

Every object in Python belongs to one of the **three generations of objects**.

When a new object is created, it gets assigned to the first generation (generation 0, because we count from 0). When the generational garbage collection runs and does not remove the object, it gets promoted to the next generation. The third (generation 2) generation is final, and the object stays there if it has been reached.

## When the generational garbage collector is running

To configure the generational garbage collection, we can specify **the maximal number of allocated objects in the first generation** and the threshold of garbage collections for the subsequent generations.

**For the second and third generation, we specify the number of garbage collections which must occur without processing those generations** to force the garbage collection of those generations.

When the number of objects in the first generation exceeds the threshold, the garbage collection runs, removes the unreachable objects, and promotes the surviving objects to the next generation.

It is important to remember that **when the threshold of the first generation gets exceeded the garbage collector processes only objects in the first generation** (the other two are not processed).

When the second threshold is exceeded, the garbage collector processes both the first and the second generation ob objects.

If the threshold of the final generation gets exceeded, the garbage collector processes all objects (such an event is called **full garbage collection**), but that does not happen every time.

The final generation has one additional threshold. **The full GC occurs only when the total number of objects in the first and second generation exceeds 25% of all objects in memory**. The additional threshold is not configurable and was added to Python to avoid running the full garbage collection too often.

## Why do we use generations of objects?

We could scan all objects every time, but the execution of the program must be stopped when the garbage collector is running, so Python programs would run significantly slower.

Fortunately, in most of the applications, **the majority of objects are "short living"** which means that they get created, are used for a while and get removed from memory during the first generation garbage collection. They don't "live" long enough to get promoted to the next generation. 

# Variable scopes

To fully describe the reference counting and the concept of "reachability," I should also write about the variable scopes in Python.

When you start a Python program, the **global scope** is created. Every variable created in that scope is called a global variable. Every object, function, method, loop, if statement, try/catch block defines a new scope. The new scope has access to all variables defined in the enclosing scope (regardless of nesting level).

**When the program reaches the end of the scope, it removes all references created in that scope. If some reference count reaches zero, the memory used by those values gets deallocated.**

# Why does it matter in Jupyter Notebook

**In Jupyter notebook, every cell uses the global scope**. Every variable you create in that scope will not get deallocated unless you override the value of the variable or explicitly remove it using the "del" keyword.

Because of that, it is easy to **waste a lot of memory if you create variables for intermediate steps in the data processing**.

In the following example, I load a data frame from a file. Make a few modifications, but every one of them is stored in a separate variable which continues to exist after I get the final result.

```python
import pandas as pd

data = pd.read_csv('file.csv')

no_nans = data.dropna()
one_hot_encoded = pd.get_dummies(no_nans)
# some other temp variables
processed_data = #...

del data
del no_nans
del one_hot_encoded
```

## Memory leaks

In programming, the situation in which a variable stops being used, but it stays in memory is called a memory leak. We can avoid memory leaks in Jupyter Notebook by **removing the temporary variables when we no longer need them**:

```python
import pandas as pd

data = pd.read_csv('file.csv')

no_nans = data.dropna()
one_hot_encoded = pd.get_dummies(no_nans)
# some other temp variables
processed_data = #...

del no_nans
del one_hot_encoded
```

**That method is not recommended**, because we can easily overlook a variable that should be removed or remove a used variable by mistake.

The better method of avoiding memory leaks is **doing data processing inside a function. It creates a new scope for the intermediate variables and removes them automatically when the interpreter exits the function**:

```python
import pandas as pd

def data_preprocessing(raw_data):
    no_nans = data.dropna()
    one_hot_encoded = pd.get_dummies(no_nans)
    # some other temp variables
    processed_data = #...
    return processed_data

data = pd.read_csv('file.csv')
processed_data = data_processing(data)
```

# See also

* GC module documentation: <a href="https://docs.python.org/3/library/gc.html" target="_blank">https://docs.python.org/3/library/gc.html</a>

* The first proposal of adding a second threshold to prevent running full GC too often: <a href="https://mail.python.org/pipermail/python-dev/2008-June/080579.html" target="_blank">https://mail.python.org/pipermail/python-dev/2008-June/080579.html</a>

* "The Garbage collector" article at PythonInternal: <a href="https://pythoninternal.wordpress.com/2014/08/04/the-garbage-collector/" target="_blank">https://pythoninternal.wordpress.com/2014/08/04/the-garbage-collector/</a>

