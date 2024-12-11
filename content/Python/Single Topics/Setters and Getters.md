---
date: 2024-07-27
modified: 2024-10-20T19:30:48+02:00
---
*These notes are derived from an excellent articles on [realpython.com](https://realpython.com/), specifically covering setters, getters, and, only partially, properties. The article is [Getters and Setters](https://realpython.com/python-getter-setter/).*

# Introduction
* In languages like Java and C++, **getter** and **setter** methods are used to write attribute in a class and they allow to access and mutate **private** ones while maintaining **encapsulation**.
* In Python, you'll typically expose attributes as part of your **Public API** and use **properties** when you need attributes with functional behaviour (such as validation). Even though properties are the Pythonic way to go, in some cases **getters** and **setters** are a preferable way to go.
# Getting to Know Getter and Setter Methods
When you want to manage class attributes, you have two ways:
* **access and mutate** the attributes directly;
* use **methods** to mutate internal data and behaviours.

If you expose the attributes of a class to your users and give the users the faculty of mutate them, then they'll be **public attributes** and they become part of the class's **public API**. Having an attribute that's part of a class's API will become a problem if you need to **change the internal implementation of the attribute itself**. An example is when you want to **turn a stored attribute into a computed one** (like for **validation** purposes): this require converting the attribute into a method, which will probably break the code because we need to change attribute access into method calls:
```python
class Circle:
	def __init__(self, radius):
		self.radius = radius

circle1 = Circle(30)
print(circle1.radius) # Output: 30
```
Adding a validation process:
```python
class Circle:
	def __init__(self, radius):
		if radius > 0:
			self._radius = radius
		else:
			raise ValueError("Radius must be greater than 0!")
	
	def radius(self):
		return self._radius
	
circle1 = Circle(30)
print(circle1.radius()) # Output: 30
```
Note that after adding the validation, you need to call `circle1.radius()` instead of `circle1.radius`  to access the attribute. This will force you to change this part in the entire code.

> [!info]- Some alternative (wrong) implementations
> Note that we could write the class as:
> ```python
> class Circle:
> 	def __init__(self, radius):
> 		if radius > 0:
> 			self.radius = radius
> 		else:
> 			raise ValueError("Radius must be greater than 0.")
> ```
> It works like the code above, but it doesn't respect the **Encapsulation** principle.
> 
> Note also that we could also write the class in this way too:
> ```python
> class Circle:
> 	def __init__(self, radius):
> 		if radius > 0:
> 			self.radius = radius
> 		else:
> 			raise ValueError("Radius must be greater than 0!")
> 			
> 	def radius(self):
> 		return self.radius
> ```
> but the `radius` method and the `radius` attribute have the same name. This causes a conflict when you try to call `circle1.radius()`.

Solution: getter and setter methods provides methods for manipulating the attributes of your classes.
# What are Getter and Setter Methods?
They're very popular in OOP languages:
+ **getter**: a method that allows you to **access** an attribute in a given class;
+ **setter**: a method that allows you to **set** or **mutate** the value of an attribute in a class.
>[!tip] When to use getter and setter methods
>In OOP, **public attributes** should be used only when you're sure that no one will ever need to attach behaviour to them. Instead, if an attribute is likely to change its internal implementation, then you should use **getter** and **setter** methods.

Implementing the getter-setter pattern requires:
1. making your **attributes non-public**;
2. writing getter and setter methods for each attribute.

Example:
```python
class Label:
    def __init__(self, text, font):
        self._text = text
        self._font = font
    
    def get_text(self):
        return self._text
    
    def set_text(self, new_text):
        self._text = new_text

    def get_font(self):
        return self._font
    
    def set_font(self, new_font):
        self._font = new_font


label = Label("Fruits", "JetBrains Mono NL")
print(label.get_text()) # Output: Fruits
print(label.get_font()) # Output: JetBrains Mono NL

label.set_text("Vegetables")
print(label.get_text()) # Output: Vegetables
```
`Label` class hides its attributes from public access and exposes getter and setter methods instead that you can use to access and mutate those attributes, which are non-public and therefore not part of the class API.
 
>[!caution] Access Modifiers in Python
>Python doesn't have the notion of **Access Modifiers**,  such as private, protected, and public, to restric access to attributes and methods in a class. In Python, the distinction is between **public** and **non-public** class members.
>
>The Python convention for non-private attributes or methods is by prefixing them with `_`. It's just a convention, so you can actually access an attribute as `obj._attribute`, but it's a **bad practice**.

# Where Do Getter and Setter Methods Come From?
Two reasons why getter and setter methods have been introduced we are partially mentioned above:
1. add validations, or transformations, or any kind of process before storing an attribute. This implies a **breaking change in the API** because it would require to add behaviour through methods, so accessing an attribute is performed with `obj.get_attribute()` and no longer `obj.attribute`. Actually, Python offers the **property** concept that we can leverage; however, we are focusing on general object-oriented programming (OOP) principles rather than specifics of Python. That’s why these languages encourage you _never to expose your attributes as part of your public APIs_. Instead, you must _provide getter and setter methods_, which offer a quick way to change the internal implementation of your attributes without changing your public API.
2. **Encapsulation** is another fundamental topic related to the origin of getter and setter methods. Essentially, this principle refers to bundling data with the methods that operate on that data. This way, access and mutation operations will be done through methods exclusively. The principle also has to do with restricting direct access to an object’s attributes, which will prevent exposing implementation details or violating state invariance.

Let's continue the `Label` class example:
```python
class Label:
    def __init__(self, text, font):
        self.set_text(text)
        self._font = font

    def get_text(self):
        return self._text
    
    def set_text(self, text_value):
        self._text = text_value.upper()


label1 = Label("Fruits", "JeyBrains Mono NL")
print(label1.get_text()) # Output: FRUITS

label1.set_text("Vegetables")
print(label1.get_text()) # Output: VEGETABLES
```

Getter and setter pattern is quite common many programming language, but not in Python. Python has a specific way to **add behaviour to an attribute without causing breaking change in the public API**, called **Properties**.
# Using Properties instead of Getters and Setters: the Python way
> [!info] Python *Properties*
> **Properties** is the Pythonic way to create methods (so that I can add behaviour, like validation, etc.) that behave like attributes (so that I can call the attribute with `obj.attribute` and no longer with `obj.get_attribute()`).

You can use properties in the same way that you use regular attributes. When you access a **property**, its attached **getter** method is automatically called. Likewise, when you mutate the property, its **setter** method gets called. This behavior provides the means to **attach functionality to your attributes without introducing breaking changes in your code’s API**.

Let's suppose you have a plain class:
```python
class Employee:
	def __init__(self, name, birth_date):
		self.name = name
		self.birth_date = birth_date
```
Now you are asked to add new requirements:
+ store `name` attribute in uppercase;
+ turn `birth_date` attribute into a `date` object.
We can do that by using properties without breaking the API:
```python
from datetime import date

class Employee:
	def __init__(self, name, birth_date):
		self.name = name
		self.birth_date = birth_date
	
	@property
	def name(self):
		return self._name

	@name.setter
	def name(self, value):
		self._name = value.upper()

	@property
	def birth_date(self):
		return self._birth_date

	@birth_date.setter
	def birth_date(self, value):
		self._birth_date = date.fromisoformat(value)
		

employee = Employee("simone", "2025-01-01")
print(employee.name) # Output: SIMONE
print(type(employee.birth_date)): # Output: <class 'datetime.date'>

employee.name = "Fabio"
print(employee.name) # Output: FABIO
```
+ `name` and `birth_date` attributes are converted into properties using the `@property` decorator;
+ now these attributes has a getter and a setter method with the name of the attribute.

As described in the learning notes specifically about [Properties](Python/Single%20Topics/Properties.md), properties have a lot of potential use cases, such as creating read-only, read-write, and write-only attributes in an elegant way.
