---
date: 2026-06-23
modified: 2026-07-27T23:39:54+02:00
---
# Why Games Make Good Teaching Examples
Games appear frequently in programming tutorials for a specific reason: they have a fixed, universally understood model. Everyone agrees on the rules, the win conditions, and the structure of play. That shared understanding means language features can be demonstrated concisely against a backdrop that requires no explanation. Battleship is a particularly good candidate for showing how object orientation appears — and how it can be avoided — in Python.

# A Review of Object Orientation
The canonical presentation of object orientation centers on **encapsulation**: bundling state together with the behavior that operates on that state, and discouraging direct manipulation of the state from outside the object.
```python
class Counter:
    def __init__(self):
        self._state = 0

    def increment(self):
        self._state += 1

    def decrement(self):
        self._state -= 1
```

The outside world is expected to call `increment` and `decrement` rather than assign to `_state` directly. Python enforces this convention weakly — the leading underscore is a signal, not a lock — and in practice teams often accept that someone might touch `_state` for a one-off reset or for debugging. Stronger encapsulation can be achieved with properties or closures, but the pattern above represents what most Python programmers mean when they say "object orientation."

# Everything in Python Is an Object
One complicating factor in discussing object orientation in Python is that the language is already thoroughly object-oriented at the interpreter level. The integer literal `0` has methods (`(0).bit_length()`). Functions have attributes (`..__name__`, `.__defaults__`, `.__code__`). Every entity the interpreter can interact with is, at the CPython level, a `PyObject` pointer. This is unlike languages such as Java, which distinguish between primitive value types and reference types. In Python, that distinction does not exist.

This means asking whether it is possible to avoid object orientation in Python is genuinely complicated. The answer depends on what you mean by the term.

# Where Is the Line?
Consider a spectrum of ways to represent the same counter, ordered from least to most object-oriented:
```python
# 1 — raw value
state = 123

# 2 — tuple
state = (increment, decrement, 123)

# 3 — named tuple
from collections import namedtuple
Counter = namedtuple('Counter', ['increment', 'decrement', 'state'])

# 4 — dataclass
from dataclasses import dataclass

@dataclass
class Counter:
    state: int = 0

# 5 — full class
class Counter:
    def __init__(self):
        self.state = 0
    def increment(self): ...
    def decrement(self): ...
```

Options 4 and 5 are unambiguously object-oriented — the `class` keyword is present, and `@dataclass` is simply a metaprogramming tool that eliminates the boilerplate from option 5 without changing its nature. Option 3 sits at the boundary: a `namedtuple` does generate a new type with `__init__`, `__repr__`, and the rest, but because it looks syntactically close to a plain tuple with named fields, most practitioners do not think of it as object orientation in any meaningful sense. Options 1 and 2 — using raw values or plain built-in types — represent clearly non-object-oriented structure, even though tuples, lists, and strings are themselves objects with methods. When someone advises "don't use an object-oriented approach here," they do not mean "avoid built-in types." They mean: write functions that accept data and return values, rather than encapsulating data inside objects with methods.

**The practical line is around option 3.** Choosing between options 4 or 5 versus options 1 through 3 is the real decision. The presence of the `class` keyword, and the intent to add custom methods to the resulting type, marks the boundary most practitioners recognize.

# Closures and Objects Are Two Sides of the Same Coin
A less obvious consequence of Python having first-class functions with closures is that the primary motivation for writing a `class` — encapsulating mutable state with behavior — can be achieved without one:
```python
# Classical object approach
class Counter:
    def __init__(self):
        self._state = 0
    def increment(self):
        self._state += 1
        return self._state
    def decrement(self):
        self._state -= 1
        return self._state

# Closure approach
def make_counter():
    state = 0
    def increment():
        nonlocal state
        state += 1
        return state
    def decrement():
        nonlocal state
        state -= 1
        return state
    return increment, decrement
```

These two formulations are functionally identical. The outputs of both are indistinguishable. The syntactic differences — dotted attribute access in one case, direct name access in the other — are superficial.

The correspondence runs deeper than syntax. Digging into the internals of a class instance reveals that its state is stored in a special dictionary, accessible as `instance.__dict__`, mapping attribute names to their current values. Every `self.state` lookup is, under the hood, a dictionary lookup into that mapping. Digging into a closure reveals that it carries a `__closure__` attribute — a tuple of _cell_ objects, each holding the current value of a closed-over variable. Every reference to `state` inside the closure functions is compiled to a fixed-index access into that tuple, rather than a dynamic dictionary lookup by name.

The two mechanisms represent the same information differently: a dictionary of named fields versus a tuple of positional cells. The class approach resolves attribute names dynamically at runtime; the closure approach resolves them statically at compile time, encoding the location of each variable as a bytecode index. Beyond that implementation detail, the structures are isomorphic.

This is a well-established result in programming language theory: **objects and closures are two sides of the same coin.** Each can simulate the other. Python gives you both, and the choice between them is largely a question of which makes the code clearer for a given context — not a question of what is possible.

---

_Next: Applying these ideas to model the game of Battleship._