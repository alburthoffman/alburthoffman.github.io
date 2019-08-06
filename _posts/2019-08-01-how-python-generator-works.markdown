---
layout: post
title:  "How python generator works"
date:   2019-08-01 21:15:12 +0800
categories: jekyll update
---

Generator is a pretty cool feature in python world. It can be used to provide a huge list with little memory requirement. Also it is the core block for coroutines, which will boost the io perfmance without using multi threads.

This article will focus on the internal part instead of how to use it. You can refer to other tutorials for that.

# What is generator
A generator is a function which contains yield or "yield from" in its body.

example:

```python
def even(a, b):
    if b <= a:
        return
    for i in range(a, b, 2):
        yield i
```

# Generator is an Object
Although it looks like a function, and defined as a function. But generator is actually an object inside Python.

the generator object will be return when calling the function.

```python
>>> def even(a, b):
...     if b <= a:
...         return
...     for i in range(a, b, 2):
...         yield i
...
>>> g = even(3, 7)
>>> g
<generator object even at 0x10d1d0c00>
>>>
```

## generator function
In OOP world, the function is more like a constructor. But it's not true in this case. Because it only creates and returns a new generator object, which contains the code as an property.

## standard methods
A generator object has at least three internal methods.

* ```__iter__```
* ```__next__```
* ```send```, send 'arg' into generator, return next yielded value or raise StopIteration.
* ```throw```, raise exception in generator, return next yielded value or raise StopIteration
* ```close```, raise GeneratorExit inside generator.

the first two methods makes it possible to be used in for in expression.

each time the generator function will return a new generator object, then we can use in for...in expression. or we can explicitly call send, throw and close to use it.

# Internals
## Code and Frame
Before digging into the c level of a generator object, let's introduce code and frame first.

### Code object
In Python, below entries will be compiled to an code object:

* module
* function
* class
* method
  
An code object has below attributes:

*  co_argcount
*  co_cellvars
*  co_code
*  co_consts
*  co_filename
*  co_firstlineno
*  co_flags
*  co_freevars
*  co_kwonlyargcount
*  co_lnotab
*  co_name
*  co_names
*  co_nlocals
*  co_stacksize
*  co_varnames

And attribute co_code contains the opcodes.

### Frame object
Code object is the static piece, while Frame object stores its runtime information. Simply speacking, when calling a code object, its frame object will be created, so that python interpreter can execute the opcodes and store the runtime data like stack in it.

these are attributes when u inspect it in python

* f_back
* f_builtins
* f_code
* f_globals
* f_lasti
* f_lineno
* f_locals
* f_trace
* f_trace_lines
* f_trace_opcodes

the field f_code points to related code object. while f_lasti records current opcode it is executing.

## Generator
So what's the relationship with Code/Frame objects and Generator?

## what does next function do
when calling ```next(g)```, python will try calling ```g.__next__```.

the generator's ```__next__``` function will delegate to send method.

the c code lays in genobject.c:

```c
static PyObject *
gen_iternext(PyGenObject *gen)
{
    return gen_send_ex(gen, NULL, 0, 0);
}
```

## what does send method do
the send method also delegate to gen_send_ex method.

```c
PyObject *
_PyGen_Send(PyGenObject *gen, PyObject *arg)
{
    return gen_send_ex(gen, arg, 0, 0);
}
```

the gen_send_ex function will push arg into the stack of frame, and then call PyEval_EvalFrameEx to execute the frame.

## what does yield do
the yield acts like return. it will be compiled to opcode *YIELD_VALUE*.

*YIELD_VALUE* simply gets the value and exit PyEval_EvalFrameEx function. While return will clean up object in the stack, and release the frame object.

So when yield happens, the frame object will keep existing in memory for later usage.

## what does "yield from" do
*yield from* is another key word used in generator. the purpose of it is to get value from another generator.

If no yield from, then we will have to call next(g) or g.send() directly, and then yield the value. which is ugly.

It will be compiled to opcode *YIELD FROM*, which will call the send method of target generator. 

After getting value from target generator, then it will act like a normal yield.

## the whole picture
Let's put all things together.

When calling a function which contains yield or yield from, it will return a generator object. Which is done by adding a if checks in PyEval_EvalFrameEx.

the function body will be compiled as gi_code of the generator object.

When calling next or send method, an frame object will be created and stored as gi_frame in the object. at first time, the frame object will stop at yield or yield from.

When calling next or send again, the gi_frame object can be reuse and passed to PyEval_EvalFrameEx. it basically will stop at yield or yield from.

When the code object returns or throws StopIterator exception, gi_frame will be released.

# when to use generator
As you can see, generator is simply a way which simplifys creating an iterator.

The cool part is we can write a simple function, and all these magics will work. Much much faster than creating a class with all these methods.

## simple
You can define an function which returns an infinite even number, and without worrying it will stack overflow.

```python
def even(start):
        if start & 0x1 == 1:
                start += 1
        while True:
                yield start
                start += 2
```

## memory
A generator saves lots of memoty as the above example shows.

As we can see from the above examples, a generator is actually a powerful tool to generate a list of values, which its size is finite or infinite.