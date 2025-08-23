---
date: 2024-07-29
modified: 2025-02-04T21:35:51+01:00
---

*These notes are derived from an excellent articles on [realpython.com](https://realpython.com/), specifically covering **properties**. The article is [Properties](https://realpython.com/python-property/).*

# Managing Attributes in your classes
Typically, there are two ways to manage attributes in classes:
+ access and mutate them directly;
+ use methods.

If you expose your attributes to the users, then they become part of the **public API** of your class. The user will access and mutate them directly in their code. The problem comes when you need to change the internal implementation of a given attribute.

Some programming languages, such as Java and C++, encourage you to never your attributes to avoid problems. Instead, you should provide [setters and getters](Setters%20and%20Getters.md) methods. These methods offer a way to change the internal implementation of your attributes without changing your public API.
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

This code is different from the getter and setter methods approach, but it looks **more Pythonic** and clean because we don't need to use methods like `_get_radius()` or `_set_radius()` anymore.

The main aspects of using `@property` as a decorator:
* The decorator approach for creating properties requires defining a first method using the public name for the underlying managed attribute, which is `.radius` in this example. This method should implement the getter logic. In the above example, lines 5 to 9 implement that method.
* When you decorate the second `.radius()` method with `@radius.setter` on line 11, you create a new property and reassign the class-level name `.radius` from line 6 to hold it. This new property contains the same set of methods of the initial property on line 6 with the addition of the new setter method provided on line 12. Finally, the decorator syntax reassigns the new property to the `.radius` class-level name.

Let's see an example:
```terminal
>>> circle = Circle(42.0)
>>> circle.radius
Get radius
42.0

>>> circle.radius = 100.0
Set radius
>>> circle.radius
Get radius 100.0

>>> del circle.radius
Delete radius
>>> circle.radius
Get radius
Traceback (most recent call last):
    ...
AttributeError: 'Circle' object has no attribute '_radius'
```

First, note that you don’t need to use a pair of parentheses for calling `.radius()` as a method. Instead, you can access `.radius` as you would access a regular attribute, which is the primary purpose of properties. They allow you to treat methods as attributes.

Here’s a recap of some important points to remember when you’re creating properties with the decorator approach:
- The `@property` decorator must decorate the **getter method**.
- The docstring must go in the **getter method**.
- The **setter and deleter methods** must be decorated with the name of the getter method plus `.setter` and `.deleter`, respectively.
# Use cases of Properties
## Providing Read-Only Attributes
The most common use of `property()` is to provide **read-only attributes** in your classes. This means that users cannot mutate the original value of these attributes:
```python
class Point:
    def __init__(self, x, y):
        self._x = x
        self._y = y

    @property
    def x(self):
        return self._x

    @property
    def y(self):
        return self._y
```

If we try to mutate these values, we get an `AttributeError`:
```terminal
>>> point = Point(12, 5)
>>> point.x
12
>>> point.y
5

>>> point.x = 42
Traceback (most recent call last):
    ...
AttributeError: can't set attribute 
```

You can also provide an explicit setter method that raises a custom exception with more specific messages:
```python
class WriteCoordinateError(Exception):
    pass

class Point:
    def __init__(self, x, y):
        self._x = x
        self._y = y

    @property
    def x(self):
        return self._x

    @x.setter
    def x(self, value):
        raise WriteCoordinateError("x coordinate is read-only")

    @property
    def y(self):
        return self._y

    @y.setter
    def y(self, value):
        raise WriteCoordinateError("y coordinate is read-only")
```

Then:
```terminal
>>> point = Point(12, 5)
>>> point.x
12
>>> point.y
5

>>> point.x = 42
Traceback (most recent call last):
    ...
WriteCoordinateError: x coordinate is read-only
```
## Creating Read-Write Attributes
To create **read-write attributes** we need to provide the getter ("read") and setter ("write") methods:
```python
class Circle:
    def __init__(self, radius):
        self.radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        self._radius = float(value)
```

We created a `Circle` class with a read-write `.radius` property. Here's how it works:
1. **Instance Initialization (`__init__` Method)**
	- When a `Circle` object is created, `__init__` takes `radius` as an argument.
	- Instead of directly assigning `self._radius = radius`, it assigns `self.radius = radius`, which triggers the `radius` setter (`@radius.setter).
2. **Getter Method (`@property`)**
	- This method acts as a getter.
	- When `circle.radius` is accessed, this method is automatically called, returning `_radius`.
3. **Setter Method (`@radius.setter`)**
	- This method acts as a setter.
	- When `circle.radius = 10` is assigned, this method is automatically called.
	- The `float(value)` ensures that the radius is always stored as a float.

This new implementation of `Circle` has a subtle detail that you should note. In this case, the class initializer assigns the input value to the `.radius` property directly instead of storing it in a dedicated **non-public attribute**, such as `._radius`. Why? Because you must ensure that **every radius value—including the initial one—goes through the setter method and gets converted to a floating-point number**.
## Providing Write-Only Attributes
You can also create **write-only** attributes by tweaking the getter method of properties. For example, you can make your getter method raise an exception every time a user accesses the underlying attribute:
```python
import hashlib
import os

class User:
    def __init__(self, name, password):
        self.name = name
        self.password = password

    @property
    def password(self):
        raise AttributeError("Password is write-only")

    @password.setter
    def password(self, plaintext):
        salt = os.urandom(32)
        self._hashed_password = hashlib.pbkdf2_hmac(
            "sha256", plaintext.encode("utf-8"), salt, 100_000
        )
```

Then:
```terminal
>>> from users import User

>>> john = User("John", "secret")

>>> john._hashed_password
b'b\xc7^ai\x9f3\xd2g ... \x89^-\x92\xbe\xe6'

>>> john.password
Traceback (most recent call last):
    ...
AttributeError: Password is write-only
```

# Putting Python’s `property()` Into Action
Let's see some practical common use cases of `property()`.
## Validating Input Values
**Data Validation** is a common requirement in code that takes input from users or other sources. For example:
```python {12-16,22-28}
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    @property
    def x(self):
        return self._x

    @x.setter
    def x(self, value):
        try:
            self._x = float(value)
            print("Validated!")
        except ValueError:
            raise ValueError('"x" must be a number') from None

	@property
    def y(self):
        return self._y

    @y.setter
    def y(self, value):
        try:
            self._y = float(value)
            print("Validated!")
        except ValueError:
            raise ValueError('"y" must be a number') from None
```

The setter methods of `.x` and `.y` use `try...except` blocks that validate input data using the Python EAFP style. It’s important to note that assigning the `.x` and `.y` properties directly in `.__init__()` ensures that the validation also occurs **during object initialization**. Here's how it works:
```terminal
>>> point = Point(12, 5)
'Validated'

>>> point.x
42

>>> point.x = 'one'
Traceback (most recent call last):
     ...
ValueError: "x" must be a number 
```

The implementation of `Point` uncovers a fundamental weakness of `property()`: we have repetitive code and this repetition breaks the **DRY** (**Don't Repeat Yourself**) principle. To avoid it, we can abstract out the repetitive logic using a **descriptor** that we can call `Coordinate`:
```python
class Coordinate:
    def __set_name__(self, owner, name):
        self._name = name

    def __get__(self, instance, owner):
        return instance.__dict__[self._name]

    def __set__(self, instance, value):
        try:
            instance.__dict__[self._name] = float(value)
            print("Validated!")
        except ValueError:
            raise ValueError(f'"{self._name}" must be a number') from None

class Point:
    x = Coordinate()
    y = Coordinate()

    def __init__(self, x, y):
        self.x = x
        self.y = y
```
## Providing Computed Attributes
Property can be used if we need an **attribute that builds its value dynamically whenever we access it**. These kinds of attributes are commonly known as **computed attributes** and they're handy when we need something that works like an *eager* attribute, but we want it to be lazy. The main reason for creating **lazy attributes** is to postpone their computation until the attributes are needed, which can make the code more efficient. Here's an example:
```python {6-8}
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    @property
    def area(self):
        return self.width * self.height
```

In this implementation, the read-only property, `.area`, computes and returns the area of the current rectangle every time you access it.

Another cool use case of properties is to provide a **formatted value** for a given attribute:
```python
class Product:
    def __init__(self, name, price):
        self._name = name
        self._price = float(price)

    @property
    def price(self):
        return f"${self._price:,.2f}"
```
## Caching Computed Attributes
When we have a computed attribute that use frequently, it's unnecessary and expensive compute it every time. In this case we can **cache the computed value** for later reuse.

If you have a property that computes its value from **constant input values**, then the **result will never change** (this means that if you change the value of `.radius`, then `.diameter` will not return a correct value):
```python
class Circle:
    def __init__(self, radius):
        self.radius = radius
        self._diameter = None

    @property
    def diameter(self):
        if self._diameter is None:
            sleep(0.5)  # Simulate a costly computation
            self._diameter = self.radius * 2
        return self._diameter
```

If you have a property that computes its value from **changeable input values**, then the result will change and we need to recalculate its value:
```python {7-9,11-14}
from time import sleep

class Circle:
    def __init__(self, radius):
        self.radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        self._diameter = None
        self._radius = value

    @property
    def diameter(self):
        if self._diameter is None:
            sleep(0.5)  # Simulate a costly computation
            self._diameter = self._radius * 2
        return self._diameter
```

In this implementation, the setter method of the `.radius` property resets `._diameter` to `None` every time you change the radius. With this little update, `.diameter` recalculates its value the first time you access it after every mutation of `.radius`.
## Logging Attribute Access and Mutation
If you ever need to **keep track of how and when you access and mutate a given attribute**, you can use a combination of `property()` and logging from `logging` module:
```python
import logging

logging.basicConfig(
    format="%(asctime)s: %(message)s",
    level=logging.INFO,
    datefmt="%H:%M:%S"
)

class Circle:
    def __init__(self, radius):
        self._msg = '"radius" was %s. Current value: %s'
        self.radius = radius

    @property
    def radius(self):
        logging.info(self._msg % ("accessed", str(self._radius)))
        return self._radius

    @radius.setter
    def radius(self, value):
        try:
            self._radius = float(value)
            logging.info(self._msg % ("mutated", str(self._radius)))
        except ValueError:
            logging.info('validation error while mutating "radius"')
```
* The getter method generates log information every time you access `.radius` in your code.
* The setter method logs each mutation that you perform on `.radius`.
* It also logs those situations in which you get an error because of bad input data.

Here's an example:
```terminal
>>> circle = Circle(42.0)
>>> circle.radius
14:48:59: "radius" was accessed. Current value: 42.0
42.0

>>> circle.radius = 100
14:49:15: "radius" was mutated. Current value: 100

>>> circle.radius
14:49:24: "radius" was accessed. Current value: 100
100

>>> circle.radius = "value"
15:04:51: validation error while mutating "radius"
```