---
date: 2026-06-23
modified: 2026-06-23T15:18:16+02:00
---
# A Subtlety in List Equality
A naïve implementation of list equality in pure Python might look like this: check that both lists have the same length, then iterate through them pairwise and verify that each corresponding pair of elements is equal.
```python
def __eq__(self, other):
    if len(self) != len(other):
        return False
    for x, y in zip(self, other, strict=True):
        if x != y:
            return False
    return True


xs = [1, 2, 3]
ys = [1, 2, 3]
    
print(
	f'{xs == ys =}',           # True
	f'{__eq__(xs, ys) =}',     # True
	sep='\n'
)
```

CPython's actual list equality implementation adds one performance optimization: before applying the `!=` check to a pair of elements, it first checks whether they are _identical_ — **whether they are the same object**. If they are identical, it skips the equality check and moves on to the next pair. This fast-path short-circuit matters because evaluating `__eq__` may invoke arbitrary Python code, while an identity check is a single pointer comparison.

The practical consequence becomes visible when a list contains a non-reflexive object — one that is not equal to itself. The canonical example is `float('nan')`:
```python
def __eq__(self, other):
    if len(self) != len(other):
        return False
    for x, y in zip(self, other, strict=True):
	    # if x is y:
		# 	continue
        if x != y:
            return False
    return True

z = float('nan')
xs = [1, 2, 3, z]
ys = [1, 2, 3, z]
    
print(
	f'{xs == ys =}',           # True
	f'{__eq__(xs, ys) =}',     # False
	sep='\n'
)
```

In the naïve implementation, `nan != nan` would cause the comparison to return `False`. In CPython's implementation, both lists hold references to the _same_ `nan` object (due to interning of the literal), so the identity check passes and the element is considered equal without invoking `__eq__` at all. The result is `True`. If line 5 and 6 would be uncommented out, we would have `f'{__eq__(xs, ys) =}'` as `True` as well.

**This behavior differs materially from NumPy.** A NumPy ndarray equality comparison is a broadcasted element-wise operation: each pair of elements is compared via `__eq__`, producing an array of booleans. A subsequent `all()` reduction on that array will return `False` if any element is `nan`, because `nan != nan` by the IEEE 754 standard. Two NumPy arrays containing `nan` at the same position are therefore never considered equal under `==`, while two Python lists in the same situation are:
```python
from numpy import array

z = float('nan')
xs = [1, 2, 3, z]
ys = [1, 2, 3, z]

assert xs==ys


z = float('nan')
xs = array([1, 2, 3, z])
ys = array([1, 2, 3, z])

assert not(xs==ys).all()
```

This difference reflects the different roles these structures play. A NumPy ndarray typically holds numerical data on which you will perform computation; element-wise comparison with full IEEE semantics is the appropriate behavior. A Python list typically holds a collection of objects to be processed; if you are iterating over two lists that contain the same objects, the fact that one of those objects is formally non-reflexive may be irrelevant to your purpose.

# Why Identity and Equality Matter in Practice
Identity and equality may seem like theoretical curiosities, but their primary practical value is in **debugging**. When a program exhibits "spooky action at a distance" — a mutation in one part of the code inexplicably affecting a completely different part — the question to ask is: are these two names referring to the same object? If they are identical, a mutation through one name is a mutation through both. Recognizing this pattern, and knowing the tools to investigate it (`is`, `id()`), is often the fastest path to diagnosing this class of bug.

# Live Views and Snapshots
When two names refer to the same object, one of them can be thought of as a **live view** of the other: any change to the underlying object is immediately reflected through both names. When one name holds an independent copy of the data as it existed at a particular moment, that is a **snapshot** — subsequent mutations to the original have no effect on it.

Neither is inherently correct. The right choice depends on what the data represents:
- **Employee permissions** should hold a live view of the employee record. If Alice is promoted, the permissions system should reflect her current role without any explicit update — there should be a single source of truth.
- **Historical payroll records** should hold a snapshot. Alice's promotion does not alter what the company paid in 2020; that record is fixed at a point in time.

Deliberately choosing between a live view and a snapshot, rather than getting one accidentally, is one of the more important design decisions in Python code that handles shared mutable state.

# Ways to Make a Copy
Python offers several syntactic forms for creating a copy of a container, and it is worth being able to recognize all of them:
```python
from copy import copy

ys = [*xs]          # unpacking into a new list — shallow copy
ys = list(xs)       # value constructor — shallow copy
ys = xs.copy()      # explicit copy method — shallow copy
ys = copy(xs)       # copy module — shallow copy
```

Any of these produces a new, independent list object whose initial contents are the same as `xs`. A subsequent `xs.append(...)` will not be visible through `ys`. What they all share is that they are **shallow**: only the outermost container is duplicated. The elements inside are not copied; they remain shared references to the same underlying objects.

# Shallow Copies vs. Deep Copies
The distinction between shallow and deep copies only matters when the container holds other mutable objects — nested lists, mutable objects, and so on.
```python
xs = [[1, 2], [3, 4]]
ys = xs.copy()          # shallow copy

xs[0].append(99)        # mutates the inner list
print(ys[0])            # [1, 2, 99] — visible through ys

xs.append([5, 6])       # mutates the outer list
print(ys)               # [[1, 2, 99], [3, 4]] — not visible through ys
```

A shallow copy duplicates the outer container, so appending to `xs` does not affect `ys`. But the inner lists are shared references, so mutating an inner list is visible through both `xs` and `ys`.

A deep copy, produced by `copy.deepcopy()`, recursively traverses the entire structure, copying every level until it reaches immutable objects. The result is a completely independent tree of objects; no mutation at any depth is visible across the two copies.

There is an inherent asymmetry here worth noting explicitly. Given a nested container, four combinations of visibility seem possible for an outer-level mutation and an inner-level mutation:

||Outer mutation visible|Inner mutation visible|
|---|---|---|
|Live reference|✓|✓|
|Shallow copy|✗|✓|
|Deep copy|✗|✗|
|_(impossible)_|✓|✗|

The fourth row cannot exist. Making an inner mutation invisible while keeping an outer mutation visible would require copying the inner structure without copying the outer one — there is no standard copy mechanism that produces this result. The asymmetry is a structural consequence of how Python's reference model works.

Shallow copies are a significant source of bugs precisely because a developer who intended a deep copy — to fully decouple two pieces of data — may have inadvertently made a shallow copy, leaving the two structures still linked at the inner level.

# Live Views Beyond Containers
The snapshot/live-view distinction extends beyond explicit data structures. A notable example is dictionary key access across Python versions. In Python 2, `dict.keys()` returned a new list — a snapshot of the keys at that moment. In Python 3, `dict.keys()` returns a **view object** — a live view that reflects subsequent additions and deletions:
```python
d = {'a': 1, 'b': 2}
keys = d.keys()       # live view in Python 3
d['c'] = 3
print(list(keys))     # ['a', 'b', 'c'] — 'c' is visible
```

To obtain a snapshot of the keys at a specific moment, use unpacking:
```python
keys = [*d.keys()]    # snapshot — future changes to d are not reflected
```

The same principle applies to module-level bindings. There are two styles of importing a name from a module:
```python
from math import sin, cos    # early binding — snapshot at import time
import math                   # late binding — live view via attribute lookup
```

The early-bound form captures whatever `sin` and `cos` refer to at the moment the `import` statement executes. If something subsequently rebinds `math.sin` (replacing it with a different function, for debugging or testing purposes), code using the early-bound names will not see the change. Code using `math.sin` via the dotted lookup will, because it evaluates `math.sin` fresh each time it is called. For most code, module attributes are never rebound after import, so the distinction has no observable effect — but for attributes like `sys.path`, which programs do modify at runtime, it can matter.

More generally, this is a specific instance of a universal pattern: `obj.x` is a live lookup — it goes to the object and retrieves the current value of `x` at the moment of evaluation — while `snapshot = obj.x` followed by later use of `snapshot` is a point-in-time copy of whatever `x` was at the moment of that assignment.

# Snapshots and Live Views in Class Design
The choice between a live view and a snapshot arises naturally when designing classes that accept mutable arguments. Consider four ways to store a sequence of values passed to a constructor:
```python
class A:
    def __init__(self, values):
        self.values = values           # live view — shares the caller's object

class B:
    def __init__(self, values):
        self.values = [*values]        # snapshot — new list

class C:
    def __init__(self, values):
        self.values = values.copy()    # snapshot — copy of original type

class D:
    def __init__(self, *values):
        self.values = values           # snapshot — new tuple from *args
```

`A` stores a live reference: mutations made to the original list outside the class are immediately visible inside it. `B` and `D` store a new independent container, fully decoupled from whatever was passed in. `C` also produces a copy, but notably preserves the type of the original container.

That type preservation matters more than it might initially appear. If the caller passes a `collections.deque` with a `maxlen` set, storing it via `[*values]` produces a plain list — the bounded, first-in-first-out behavior of the deque is silently discarded. Calling `.copy()` on the deque preserves both the data and the behavioral properties. Similarly, a `collections.defaultdict` has semantically meaningful behavior (returning a default value for missing keys) that is lost if you convert it to a plain dict via `{**values}` but preserved if you call `.copy()`.

The right choice is a deliberate one:
- If external mutations to the passed-in object should be reflected inside the class, use direct assignment.
- If you want an independent copy that preserves the container's type and behavior, use `.copy()`.
- If you want an independent copy as a specific concrete type regardless of what was passed in, use the explicit constructor or unpacking syntax.

---

_Next: Hashability versus immutability._
