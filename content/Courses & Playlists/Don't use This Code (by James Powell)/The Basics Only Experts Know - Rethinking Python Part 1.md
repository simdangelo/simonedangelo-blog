---
date: 2026-06-10
modified: 2026-06-10T23:39:30+02:00
---

# Variables as Names 
In Python, **variable names** are exactly that: names. They are not memory locations holding data directly, as they would be in C or C++. A name is simply a **reference** — a **binding** — to some underlying object. This distinction matters immediately, because it means multiple names can refer to the same underlying object at the same time.

Python provides two operators for asking questions about the objects a name refers to: `==` for equality and `is` for identity.

Consider three names — `xs`, `ys`, and `zs` — where `xs` and `ys` hold lists with the same values, but are distinct objects, while `zs` is not:
```python
xs = [1, 2, 3]
ys = [1, 2, 3]
zs = [400, 500, 600]
```

The equality check (`==`) tells us that `xs` and `ys` are equivalent — they contain the same values. The identity check (`is`) tells a different story: `xs is ys` is `False`, because they are two distinct objects in memory.

These two operators answer fundamentally different questions:
* **Equality** asks whether two objects _can be used interchangeably for a given purpose_.
* **Identity** asks whether two names _refer to exactly the same object_.

# Equality Is Customizable; Identity Is Not
Equality checking in Python is fully customizable. Consider a dictionary comparison:
```python
D0 = {'a': 1, 'b': 2}
D1 = {'b': 2, 'a': 1}
D2 = {'c': 3}

D0 == D1  # True  — same keys and values, order irrelevant
D0 == D2  # False
D0 is D1  # False — distinct objects
```

Standard dictionaries consider two **dicts** equal if they **contain the same key-value pairs**, regardless of insertion order. `OrderedDict`, by contrast, requires the same order as well. Which semantics you want depends on your use case — JSON documents, for instance, treat node order as irrelevant, while XML order is part of the document structure.

The same principle applies to any user-defined class. You can implement `__eq__` to define what equality means for your objects. For example, if an object holds a `name`, a `value`, and some `metadata` capturing provenance (say, the filename from which it was loaded), you might reasonably decide that equality should compare only `name` and `value`, ignoring `metadata`. Two records with identical content are semantically equivalent regardless of where they came from.

**Identity, on the other hand, cannot be customized in any way.** The `is` operator always performs exactly one operation: it checks whether two names refer to the same object. There is no `__is__` method to implement, no hook to override, and no way to influence this behavior from Python code.

Similarly, **the assignment operator (`=`) cannot be customized.** In C++, copy constructors and move constructors allow you to define what assignment means for a type. Python has no such mechanism. `x = y` always means one thing: `x` becomes an alternate name for the same object that `y` refers to. This is a name binding — or a name _re_binding if `x` previously referred to something else — and it behaves identically regardless of any custom code associated with the objects involved.

There is one narrow and largely academic exception to this rule. During class construction, it is possible to replace the namespace dictionary in which the class body executes (via `__prepare__`) with a custom mapping whose `__setitem__` logs or reacts to assignments. This allows you to intercept statements like `x = 1` inside a class body:
![[Pasted image 20260610231907.png]]
![[Pasted image 20260610231915.png]]

This is a curiosity rather than a practical tool. If used at all, it should be for something transparent — a debugging aid or a well-understood optimization — not for anything that would surprise a reader of the code.

# The `id()` Function
Python's built-in `id()` returns a **unique integer identifier** for an object at a given point in time. In CPython, this is the **object's memory address**. The relationship between `is` and `id()` is exact:
```python
xs is zs        # True
id(xs) == id(zs)  # True  — always equivalent
```

Saying `xs is zs` is precisely the same claim as saying `id(xs) == id(ys)` **at the same moment in time**. That qualification matters, as we will see shortly.

# The Central Consequence of Identity: Shared Mutation
The most important practical consequence of two names being identical — referring to the same object — is that **a mutation performed through one name is visible through the other**. If `xs` and `zs` are identical, appending to `xs` also appears to modify `zs`, because there is only one list and both names point to it:
```python
xs = [1, 2, 3]
ys = [1, 2, 3]
zs = xs

xs.append(4)

print(xs)  # [1, 2, 3, 4]
print(zs)  # [1, 2, 3, 4]  ← same object, mutation visible
print(ys)  # [1, 2, 3]     ← independent copy, unaffected
```

`xs.append(4)` performs a mutation on the object to which `xs` refers — which is the same object to which `zs` refers. `ys`, being a separate object that merely happens to be equal, is unaffected. This is the core distinction to internalize: **identity means shared fate under mutation; equality means only shared value at a point in time.**

# Interning and Constant Folding
CPython applies several performance optimizations that can produce surprising results when inspecting object IDs. Because the primary consequence of identity is shared mutation, and because integers and strings are **immutable** (they cannot be mutated at all), Python is free to intern commonly used values — that is, allocate them only once and reuse the same object wherever they appear.

On startup, CPython pre-allocates small integers (typically −5 through 256) and may deduplicate small strings. As a result, two separately created instances of a small integer may share the same ID:
```python
print(
    f'{id(eval("123")) = :#_x}',
    f'{id(eval("123")) = :#_x}',
    f'{id(eval("123_456")) = :#_x}',
    f'{id(eval("123_456")) = :#_x}',
    sep='\n'
)

# =============Output=============
# id(eval("123")) = 0x1_0718_e3a8
# id(eval("123")) = 0x1_0718_e3a8
# id(eval("123_456")) = 0x1_04e6_32d0
# id(eval("123_456")) = 0x1_04e6_21f0
```

The `eval` calls are necessary here to prevent a related but distinct optimization: **constant folding**. When Python compiles a module, it identifies constants that appear multiple times within that _compilation unit_ and folds them into a single object. Without `eval`, every literal `123456` in the same module would already be the same object before the code even runs — which would obscure the interning behavior we want to observe. At the interactive REPL, each expression is compiled independently, so constant folding does not apply across separate inputs.

Constant folding is scoped to the compilation unit. Two functions defined in the same module will share folded constants:
```python
# module_a.py
def f():
    return 123456789

def g():
    return 123456789

f.__code__.co_consts is g.__code__.co_consts  # the constant is shared within the module
```

But the same literal in a function defined in a different module produces a distinct object. Again, **because integers are immutable, none of this has any observable effect on program behavior** — which is precisely why Python is comfortable applying these optimizations to reduce redundant memory allocations. The only situation where this detail is likely to confuse you is if you are inspecting IDs directly, or if you notice slightly different behavior between running code as a script versus typing it at the REPL.