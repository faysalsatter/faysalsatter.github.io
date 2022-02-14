---
layout: post
title: "Singletons and interning"
subtitle: "This is about how python store certain numbers and strings in its memory."
category: Python
tags: [python, pattern, objects, singletons, interns]
---

In the [previous post]({% post_url 2021-08-07-design_pattern %}), I give a short journey how singleton become an important part in software designing. Here, we will see how singleton actually behave in python using some simple code examples. Short recap: Singletons are objects that exist one and only place in memory throughout the lifetime of a program. And interning is the process of making an object a singleton. When python starts, it automatically interns some objects. Also, we can make manually intern some objects as singletons.

Note: These examples run in Cpython (v 3.8.11) Jupiter notebook environment. Since singletons and interning works differently in different python version, environment or implementation (Jython, IronPython, PyPy), it might work differently anywhere else. For example, integer singletons are disabled in PyPy by default. If we want such features, we have to enable it first.

## Memory Location

It all starts from the memory address of a python object. Everything (literally everything!!) in python is an object and every object has a location/address in the memory. We can see the unique memory location of an object using `id()` function:

```python
>>> id(100)
140729355608944

>>> id("hello")
2756760710768

>>> id(None)
140729355376768
```

If we want to check if two objects are pointing to the same memory location, we can use the `is` operator.  On the other hand, if we want to compare the value/content of two objects, we can use the `==` operator. For example,
```python
>>> a = 100
>>> b = 100

>>> id(a)
140729184166768

>>> id(b)
140729184166768

>>> a == b
True

>>> a is b
True
```
Since the values of both `a` and `b` are same (`100`) here, `a == b` is `True`. Also, `a is b` is `True`, as both objects share the same memory location (`140729184166768`). Essentially, `a is b` is same as `id(a) == id(b)`.

We will use the `==` and `is` operators to explore the singletons and interning with different data types. We will start with Integers.

## Integers
Any integers from -5 to 256 (inclusive) will be singletons objects. These numbers are already saved in the memory. Any time python call a number from this range, same same number will be used.  Lets verify this by assigning a numbers outside the range of -5 to 256, and compare if they come from same location.
```python
>>> a = -6
>>> b = -6
>>> a == b
True
>>> a is b
False

>>> a = 257
>>> b = 257
>>> a == b
True
>>> a is b
False
```
which shows that `a` and `b` are two different objects. On the contrary, if a number between -5 and 256 is assigned to different variables, they come from same memory location, making it singleton.
```python
>>> a = -5
>>> b = -5
>>> a == b
True
>>> a is b
True

>>> a = 256
>>> b = 256
>>> a == b
True
>>> a is b
True
```

## Floats
Floats do not go through interning process. Each and every float is assigned to a different memory location.
```python
>>> a = 1.0
>>> b = 1.0
>>> a == b
True
>>> a is b
False
```

## Lists
In general, two lists are two different objects.
```python
>>> a = [100]
>>> b = [100]
>>> a == b
True
>>> a is b
False
```
But if we assign a variable by another variable, it will point to the same object.
```python
>>> a = [1, 2, 3]
>>> b = a
>>> a == b
True
>>> a is b
True
```
Since these two lists are same objects, any operation on one object will have the same impact on the other.
```python
>>> a = [1, 2, 3]
>>> b = a
>>> b.append(4)

>>> a
[1, 2, 3, 4]
>>> b
[1, 2, 3, 4]

>>> a == b
True
>>> a is b
True
```
To create a different object with the same list values, use `copy`
```python
>>> import copy
>>> a = [1, 2, 3]
>>> b = copy.copy(a)
>>> b.append(4)

>>> a
[1, 2, 3]
>>> b
[1, 2, 3, 4]

>>> a == b
False
>>> a is b
False
```

## Strings
The interning rules for strings are much more complex. 

#### Empty/blank strings

Empty/blank strings of length 0 and 1 are interned.
```python
>>> a = ""
>>> b = ""

>>> len(a)
0
>>> len(b)
0

>>> a == b
True
>>> a is b
True


# One space string is of length 1. It will be interned.
>>> a = " "
>>> b = " "

>>> len(a)
1
>>> len(b)
1

>>> a == b
True
>>> a is b
True


# But two space string is of length 2. So it will not be interned.
>>> a = "  "
>>> b = "  "
>>> len(a)
2
>>> len(b)
2
>>> a == b
True
>>> a is b
False
```
#### Strings with one word
String with only one word will be interned, as long as it looks like identifier (in snake_case format), i.e., strings have ascii letters, digits or underscores.

```python
# One word string with ascii letters, digits and underscore.
>>> a = "Hello_01"
>>> b = "Hello_01"

>>> a == b
True
>>> a is b
True


# One word string with non-ascii letter (!)
>>> a = "Hello!"
>>> b = "Hello!"

>>> a == b
True
>>> a is b
False


# Two word string
>>> a = "Hello World"
>>> b = "Hello World"

>>> a == b
True
>>> a is b
False
```
#### Unicode characters 
Unicode characters from 0 to 255 are interned.

```python
# Unicode characters between 0 and 255 (inclusive) are interned.
>>> a = chr(255)
>>> b = chr(255)
>>> print(a)
ÿ

>>> a == b
True
>>> a is b
True

# Unicode character 256 will not be interned.
>>> a = chr(256)
>>> b = chr(256)
>>> print(a)
Ā
>>> a == b
True
>>> a is b
False
```

####  Compile time vs runtime
Strings are interned at compile time, not runtime.

```python
# String concatenation is done at compile time
>>> a = 'Hello_' + 'World'
>>> b = 'Hello_World'
>>> print(a)
Hello_World

>>> a == b
True
>>> a is b
True


# But string join is done at runtime
>>> a = ''.join(['H', 'e', 'l', 'l', 'o'])
>>> b = 'Hello'
>>> print(a)
Hello

>>> a == b
True
>>> a is b
False
```

#### Peephole optimized strings
Python optimize its code at compile time to increase speed and efficiency by utilizing a process named Peephole optimization. One of these optimizations is constant folding, where python pre-calculates constant expression and replace the original expression with calculated value at compile time. For example, consider the expression `x = 2 * 4 * 3`. Python compiler  identify and evaluate such constant expressions and substitute the computed values (x = 24) at compile time. Python version up to 3.6 uses peephole optimization.

So what does it do with string interning? String of length ≤ 20 gets peephole optimized. The expression `'z' * 20` internally becomes `'zzzzzzzzzzzzzzzzzzzz'` by constant folding optimization and gets interned.
```python
>>> a = 'z' * 20
>>> b = 'zzzzzzzzzzzzzzzzzzzz'

>>> a == b
True
>>> a is b
True
```

However, Python from 3.7 uses the AST optimizer, and string up to length 4096 gets interned. 
```python
# String of length 4096 gets interned
>>> a = 'z' * 4096
>>> b = 'z' * 4096

>>> a == b
True
>>> a is b
True


# Stings of length 4097 does not!
>>> a = 'z' * 4097
>>> b = 'z' * 4097

>>> a == b
True
>>> a is b
False
```

#### Names of functions, class, variables, arguments, etc. are implicitly interned.
```python
>>> def func():
...     print('Hello')

>>> name = 'func'
>>> func.__name__ is name
True
```

<!-- ## The keys of dictionaries used to hold module, class, or instance attributes are interned. -->

## Force/explicit interning
Sometimes we might need to intern some strings explicitly. One use case is interning stop words for NLP. Stop words are a set of commonly used words in a language. Stop words occur so frequently that they carry little value of analysis. Interning these stop words will reduce memory.

We can intern any lengths of strings by using `sys` module:
```python
# python2 has build-in "intern" function.
>>> from sys import intern
>>> a = intern('Hello World')
>>> b = intern('Hello World')

>>> a == b
True
>>> a is b
True
```
Remember, "Hello World" would not be interned without this force interning process.

## Bonus
If we create a Python script as:
```python
# script.py
a = 256
b = 256
print("a is b:", a is b)
x = 257
y = 257
print("x is y:", x is y)
```
and run the script as `python script.py`, guess what will print?

It will print:
```
a is b: True
x is y: True
```
Wait! `x is y` should be `False`, as `257` is outside of integer singletons range!!

Here's the fun part! When we run a script, Python compiler can see the whole code and do optimizations as necessary to save time and memory. While running the above script, Python figures out that `x` and `y` are both assigned to the same integer `257`, and there is no reason to create two separate objects. Instead, it creates only one object and assigns both variables to that object.

On the other hand, when we run code in interactive Python shell or Jupyter notebook, compiler can see only one line at a time. It will have no idea if integer `257` assigned before or not. Since `257` is not a singleton, compiler creates a new object for `257`.

So, in an interactive shell, if we assign `257` to two different variables in one statement, Python compiler will see both variables at the same time. It will do the optimization by creating only one object and assign both variables to the same object.
```python
>>> a, b = -6, -6
>>> a == b
True
>>> a is b
True
```