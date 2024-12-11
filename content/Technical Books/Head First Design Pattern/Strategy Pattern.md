---
date: 2024-06-16
modified: 2024-06-21T19:53:05+02:00
---
***Goal***: build a duck pond simulation game, where the game can show a large variety of duck species.

# Problems and Potential Solutions
---
***Solution***: Let's use standard OO techniques and created one **Duck Superclass** from which **all other duck types inherit**:

![](Technical%20Books/Head%20First%20Design%20Pattern/attachments/Design%20Pattern.drawio%20(1).svg)
* All ducks quack and swim. The superclass takes care of the implementation code;
* the `display()` method is abstract, since all duck subtypes look different;
* each subtype is responsible for implementing its own `display()` behaviour;

Now we want to provide the ducks the ability to fly. We add this new method in the `Duck` superclass:
![](Technical%20Books/Head%20First%20Design%20Pattern/attachments/Design%20Pattern.drawio%20(2).svg)

---

***Problem***: by putting `fly()` in the superclass, he gave the flying ability to ALL ducks including those that shouldn't fly. Moreover, some ducks (such as wooden decoy and rubber ducks don't even quack).

***Solution***: override `fly` and `quack` methods:
![](Technical%20Books/Head%20First%20Design%20Pattern/attachments/Design%20Pattern.drawio%20(3).svg)

---

***Problem***: you are forced to possibly override `fly()` and `quack()` for every new `Duck` subclass that's ever added to the program forever.
> [!bug] Inheritance
>**Inheritance** probably wasn't the answer.

***Solution***: we need a clearer way to have only some (but not all) of the duck types fly or quack. We can take the `fly()` and `quack()` out of the `Duck` superclass, and make `Flyable` and `Quackable` **interfaces** with respective methods so that only the ducks that are supposed to fly will implement that interface and have those methods:

![](Technical%20Books/Head%20First%20Design%20Pattern/attachments/Design%20Pattern.drawio%20(8).svg)

---
***Problem***: abstracting flying and quacking behaviour solves part of the problem (no inappropriately flying rubber ducks), but it makes the maintenance a nightmare. Indeed:
* since they're interface, you need to implement code for flying and quacking behaviour for each duck types (potential duplicated code);
* whenever you need to modify a behaviour, you're often forced to track down and change it in all the different subclasses when that behaviour is defined.

***Solution***:
> [!success] First Design Principle
> Identify the aspects of your application that **vary** and **separate** them from **what stays the same**.

Another way of thinking about this principle: **take the parts that vary and encapsulate them**, so that later you can alter or extend the parts that vary without affecting those that don't.

In our case, since we know that `fly()` and `quack()` are the parts of the `Duck` class that vary across ducks, we're going to create **two sets of classes**, one for `fly` and one for `quack`. Each set of classes will hold the **implementation of the respective behaviour**. For instance, we might have:
* a class implementing quacking
* a class implementing squeaking
* a class implementing silence
* ...

What we want to do is:
* **assign specific implementations of each behaviour** to all possible `Duck` instances;
* add the **possibility to change the implementation of a specific behaviour** of duck instances dinamically.
> [!success] Second Design Principle
> Program to an interface, not an implementation.

We'll use an **interface** to represent each behaviour (such as `FlyBehaviour` and `QuackBehaviour`), and each implementation of a behaviour will implement one of those interfaces:
![](Technical%20Books/Head%20First%20Design%20Pattern/attachments/Design%20Pattern.drawio%20(6).svg)

With this approach, the Duck subclasses will use a behaviour represented by an **interface** (`FlyBehaviour` and `QuackBehaviour`), so that the **actual implementation** of the behaviour won't be locked into the Duck subclasses.
> [!TIP] New Approach
>From now on, the Duck behaviours will live in a separate class - a class that implements a particular interface.
> 
> That way, the Duck classes **won't need to know any of the implementation details for their own behaviours**.
>
>With this design, other types of objects can reuse our fly and quack behaviours because these behaviours are no longer hidden away in our Duck classes.
>
>And we can add new behaviours without modifying any of our existing behaviour classes or touching any of the Duck classes that use flying behaviours.

# Integrating the Duck Behaviours
Here's the key: a Duck will now delegates its flying and quacking behaviours, instead of using quacking and flying methods defined in the Duck class (or subclass).
### Add instance variables of type `FlyBehaviour`and `QuackBehaviour`
Here's the new `Duck` class:
![](Technical%20Books/Head%20First%20Design%20Pattern/attachments/Design%20Pattern.drawio%20(7).svg)
* each concrete duck object will assign to `FlyBehaviour` and `QuackBehaviour` variables a specific behaviour at runtime, like `FlyWithWings`.
### Implement `performQuack` and `performFly`
```python
from abc import ABC, abstractmethod

class QuackBehaviour(ABC):
     @abstractmethod
     def quack(self):
          pass

class FlyBehaviour(ABC):
     @abstractmethod
     def fly(self):
          pass

class Duck():
    def __init__(self, quackBehaviour, flyBehaviour):
        self.quackBehaviour = quackBehaviour
        self.flyBehaviour = flyBehaviour

    def performQuack(self):
        self.quackBehaviour.quack()

    def performFly(self):
        self.flyBehaviour.fly()
```
To perform a quack (the same for fly), a Duck just asks the object that is referenced by `quackBehaviour` to `quack()` for it. So, we don't care what kind of object the concrete Duck is, all we care about is that it knows how to `quack()`.

### How the `quackBehaviour` and `flyBehaviour` instance variable are set?
Let's create concrete implementation of these behaviours and a specific type of duck called `MallardDuck`:
```python
class Quack(QuackBehaviour):
	def quack(self):
		print("Quack! Quack!")

class Squeak(QuackBehaviour):
	def quack(self):
		print("Squeak! Squeak!")

class MuteQuack(QuackBehaviour):
	def quack(self):
		print("I'm not able to quack.")

class FlyWithWings(FlyBehaviour):
	def fly(self):
		print("I'm flying with wings.")

class FlyNoWay(FlyBehaviour):
	def fly(self):
		print("I'm not able to fly.")


class MallardDuck(Duck):
    def __init__(self):
        super().__init__(quackBehaviour=Quack(), flyBehaviour=FlyWithWings())
        
    def display(self):
        print("I'm a real Mallard Duck")
```
When a `MallardDuck` is instantiated, its constructor initializes the duck with specific behaviors: it sets the `quackBehaviour` to an instance of the `Quack` class, which defines the sound "Quack! Quack!", and the `flyBehaviour` to an instance of the `FlyWithWings` class, which defines the action of flying with wings. This setup allows the `MallardDuck` to perform its specific quacking and flying behaviors when the corresponding methods are called. Let's prove it:
```python
aMallardDuck = MallardDuck()

aMallardDuck.performFly()
aMallardDuck.performQuack()
```
Output:
```
I'm flying with wings.
Quack! Quack!
```
# Setting behaviour Dinamically
It might be more useful to set the duck's behaviour type through a **setter method** on the Duck class, **rather than instantiating in in the duck's constructor**:
```python
class Duck():
    def __init__(self, quackBehaviour, flyBehaviour):
        self.quackBehaviour = quackBehaviour
        self.flyBehaviour = flyBehaviour

    def performQuack(self):
        self.quackBehaviour.quack()

    def performFly(self):
        self.flyBehaviour.fly()

    def setFlyBehaviour(self, fb):
        self.flyBehaviour = fb

    def setQuackBehaviour(self, qb):
        self.quackBehaviour = qb
```
We can call these new methods anytime we want to change the behaviour of the duck. Let's prove that by creating a new duck class:
```python
class ModelDuck(Duck):
    def __init__(self):
        super().__init__(quackBehaviour=Quack(), flyBehaviour=FlyNoWay())
        
    def display(self):
        print("I'm a real Model Duck")


aModelDuck = ModelDuck()

aModelDuck.performFly()
aModelDuck.performQuack()
```
Output:
```
I'm not able to fly.
Quack! Quack!
```
As expected, this new duck is not able to fly because its constructor sets the `flyBehaviour` to an instance of the `FlyNoWay`. But now we can dynamically change its flying behaviour:
```python
aModelDuck.setFlyBehaviour(FlyWithWings())

aModelDuck.performFly()
aModelDuck.performQuack()
```
Output:
```
I'm flying with wings.
Quack! Quack!
```
# The Big Picture on Encapsulated behaviours
Here's the entire reworked class structure:
* ducks extending `Duck` class;
* fly behaviours implementing `FlyBehaviour` and quack behaviours implementing `QuackBehaviour`
![](Technical%20Books/Head%20First%20Design%20Pattern/attachments/Design%20Pattern.drawio%201.svg)
Note that different arrows represent different relationship between objects:
* **IS-A** relationship:
	* represents **Inheritance**;
	* indicates that a subclass inherits behaviour and attributes from a superclass;
	* example: a `MallardDuck` **is-a** `Duck`;
	* it's represented by the following arrow:
	![](Technical%20Books/Head%20First%20Design%20Pattern/attachments/Design%20Pattern.drawio%20(1)%202.svg)
* **HAS-A** relationship:
	* represents **Composition**;
	* indicates that a class contains or is composed of another class;
	* example: a `Duck` **has-a** `FlyBehaviour`;
	* it's represented by the following arrow:
	![](Technical%20Books/Head%20First%20Design%20Pattern/attachments/Design%20Pattern.drawio%20(2)%201.svg)
# HAS-A is better than IS-A
With the HAS-A relationship, each duck has a `FlyBehaviour` and a `QuackBehaviour` to which delegates flying and quacking.

When you put two classes together like this you're using **Composition**. Instead of *inheriting* the behaviour, the ducks get their behaviour by being *composed* with the right behaviour object.

> [!success] Third Design Principle
> Favor **Composition** over Inheritance.

# Strategy Pattern
What we have applied is called **Strategy Pattern**.

> [!summary] Strategy Pattern Definition
> The **Strategy Pattern** defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.
