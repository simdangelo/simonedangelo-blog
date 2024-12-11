---
date: 2024-07-29
modified: 2024-10-20T19:30:15+02:00
---

*These notes are derived from an excellent articles on [realpython.com](https://realpython.com/), specifically covering **properties**. The article is [Properties](https://realpython.com/python-property/).*

# Managing Attributes in your classes
Typically, there are two ways to manage attributes in classes:
+ access and mutate them directly;
+ use methods.

If you expose your attributes to the users, then they become part of the **public API** of your class. The user will access and mutate them directly in their code. The problem comes when you need to change the internal implementation of a given attribute.

Some programming languages, such as Java and C++, encourage you to never your attributes to avoid problems. Instead, you should provide [setters and getters](Python/Single%20Topics/Setters%20and%20Getters.md) methods. These methods offer a way to change the internal implementation of your attributes without changing your public API.
>[!note]
>Note that **getter and setters** are often considered **anti-pattern** and a signal of poor object-oriented design. The main argument behind this proposition is that these methods break Encapsulation. They allow you to access and mutate the components of your objects.

In the end, these languages need getter and setter methods because they don’t provide a suitable way to change the internal implementation of an attribute if a given requirement changes. Changing the internal implementation would require an API modification, which can break your end users’ code.

# The Getter and Setter Approach in Python
Despite not being a best practice, you can use getter and setter methods in Python:
```python
class Point:
    def __init__(self, x, y) -> None:
        self._x = x
        self._y = y

    def get_x(self):
        return self._x
    
    def set_x(self, value):
        self._x = value

    def get_y(self):
        return self._y
    
    def set_y(self, value):
        self._y = value
```
Possible usage:
```bash {2,4,5}
>>> point = Point(12, 5)
>>> point.get_x()
12
>>> point.set_x(42)
>>> point.get_x()
42
```
>[!note] Public and Non-Public class members in Python
>Python doesn't have the notion of **access modifiers**, such as private, protected, and public to restrict access to attributes and methods. The Python the distinction is between **public** and **non-public** class members.
>
>If you want to sign an attribute or method as non-public, then you have to use the well-known Python convention of prefixing the name with an underscore (`_`). **This is just a convention!** Nothing stops you to access attributes and mutate them using dot notation, such as `self._x = 10`.
# The Pythonic Approach
The code we have written above doesn't look Pythonic. Instead, you can handle requirement changes without changing the public API by turning attributes into **Properties**.

Properties represent an intermediate functionality between a plain attribute and a method. In other words, they allow you to create **methods that behave like attributes**.

If you turn `.x` and `.y` into properties, you can continue accessing them as attribute, but you can modify their internal implementation. This allows you to expose attributes as part of your public API.
# Getting Started With Python’s `property()`
Python’s `property()` is the Pythonic way to avoid formal getter and setter methods as we did before, but it allows us to attach getter and setter to class attribute in a such a way you can handle the internal implementation for that attribute without exposing getter and setter methods in your API.

Here’s the full signature of `property()`:
```python
property(fget=None, fset=None, fdel=None, doc=None)
```
The first two arguments take function objects that will play the role of getter (`fget`) and setter (`fset`) methods. The return value of `property()` is the managed attribute itself. If you access the managed attribute, as in `obj.attr`, then Python automatically calls `fget()`. If you assign a new value to the attribute, as in `obj.attr = value`, then Python calls `fset()` using the input `value` as an argument. Finally, if you run a `del obj.attr` statement, then Python automatically calls `fdel()`.

You can use `property()` either as a function or a decorator to build your properties. The decorator approach is much more popular.
## Creating Attributes With `property()`
You can create a property by calling `property()` with an appropriate set of arguments and assigning its return value to a class attribute. All the arguments to `property()` are optional. However, you typically provide at least a **setter function**.
Let's rewrite the `Cirlce` class:
```python
class Circle:
    def __init__(self, radius) -> None:
        self._radius = radius

    def _get_radius(self):
        print("Get radius")
        return self._radius
    
    def _set_radius(self, value):
        print("Set radius")
        self._radius = value

    def _del_radius(self):
        print("Delete radius")
        del self._radius

    radius = property(
        fget=_get_radius,
        fset=_set_radius,
        fdel=_del_radius,
        doc="The radius property."
    )
```
We define three non-public methods:
1. **`._get_radius()`** returns the current value of `._radius`
2. **`._set_radius()`** takes `value` as an argument and assigns it to `._radius`
3. **`._del_radius()`** deletes the instance attribute `._radius`

Firstly, it creates a new attribute on the `Circle` class called `radius`. It sets this attribute to be a **property**. Under the hood, a `property` attribute delegates the real work to the three methods we created:
* when used in an access context, the first function gets the value;
* when used in an update context, the second function sets the value.

To initialize the property, you pass the three methods as arguments to `property()`.
```bash {3,6,8,11}
>>> from circle import Circle
>>> circle = Circle(42.0)
>>> circle.radius
Get radius
42.0
>>> circle.radius = 100.0
Set radius
>>> circle.radius
Get radius
100.0
>>> del circle.radius
Delete radius
>>> circle.radius
Get radius
Traceback (most recent call last):
    ...
AttributeError: 'Circle' object has no attribute '_radius'

>>> help(circle)
Help on Circle in module __main__ object:

class Circle(builtins.object)
    ...
 |  radius
 |      The radius property. 
```
* When you access the attribute with `circle.radius`, Python automatically calls `._get_radius()`;
* when you assign a value to the attribute with `circle.radius=100`, Python automatically calls `._set_radius()`;
* when you execute `del circle.radius`, Python calls `._del_radius()`, which deletes the underlying `._radius`.

Properties are **class attributes** that manage **instance attributes**. You can think of a property as a collection of methods bundled together.
## Using `property()` as a Decorator
In Python, a **decorator** is a function that take another functions as an argument and return a new function with added functionality.

The decorator syntax consists of placing the name of the decorator function with a leading `@` symbol right before the definition of the function you want to decorate:
```python
@decorator
def func(a):
	return a
```
This is equivalent to:
```python
def func(a):
	return a

func = decorator(func)
```

Python `property()` can also work as a decorator using the syntax we saw early:
```python {5,6,7,8}
class Circle:
    def __init__(self, radius) -> None:
        self._radius = radius

    @property
    def radius(self):
        print("Get radius")
        return self._radius
    
    @radius.setter
    def radius(self, value):
        print("Set radius")
        self._radius = value

    @radius.deleter
    def radius(self):
        print("Delete radius")
        del self._radius 
```
This code is different from the getter and setter methods approach, but it looks **more Pythonic** and clean because we don't need to use methods like `_get_radius()` or `_set_radius()`. The main aspects:
* the decorator approach for creating properties requires defining a first method using the public name for the underlying managed attribute, which is `.radius` in our example. This method implements the getter logic (from line 5 to 8);
* ... *I want to understand better what's happening under the hood when you use .setter. Spend more time on this topic and fill this missing space.*
