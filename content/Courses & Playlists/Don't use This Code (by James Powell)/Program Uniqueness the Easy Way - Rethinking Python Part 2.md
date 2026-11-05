---
date: 2026-06-11
modified: 2026-06-11T22:37:08+02:00
---

# What `id()` Actually Returns
In CPython, calling `id()` on an object returns the **memory address** of the underlying `PyObject` pointer — a plain C `long` cast from the pointer value. The CPython source makes this explicit:
![[Pasted image 20260611213237.png]]

That is all `id()` does: it takes whatever pointer CPython is using to represent the object internally and converts it to a Python integer. This is not guaranteed across Python implementations. In IronPython, `id()` may return a monotonically increasing counter; in PyPy, it may be something else entirely. For the overwhelming majority of practitioners using CPython, however, the ID of an object is its memory address.

There is not much you can directly _do_ with that address from pure Python — and there are deliberate safeguards against trying. Python's audit hook mechanism allows code to register a callback on certain built-in operations, and `id()` is one of them. The motivation is precisely the scenario described here: knowing an object's memory address opens the door to low-level memory manipulation that could break out of a sandbox or circumvent code-signing mechanisms.

A concrete illustration of why this matters: given an object's address, it is possible to use `numpy.lib.stride_tricks` to construct a NumPy array whose memory overlaps with that object's storage, and then write through that array to mutate bytes that Python considers read-only. The canonical demonstration is mutating a tuple — an object that Python's type system treats as immutable — by writing directly to its underlying memory. This is technically possible in CPython and mostly works, but it is not something that should appear in any real code. An audit hook watching for `id()` calls can detect and block the attempt.

# ID Reuse and Point-in-Time Uniqueness
Because `id()` returns a memory address and memory is finite, **IDs are recycled**. After an object is garbage-collected and its memory freed, CPython may allocate a new object at the same address, giving it the same ID as the original. The following illustrates this immediately:
```python
xs = [1, 2, 3]
print(f'{id(xs) = :#_x}') # id(xs) = 0x1_04b7_8700
del xs

ys = [1, 2, 3, 4]
print(f'{id(ys) = :#_x}') # id(ys) = 0x1_04b7_8700
```

`xs` and `ys` are clearly different objects, yet they share an ID. The reason is that deleting `xs` removed the last reference to it, triggering garbage collection, and CPython then reused that memory location for `ys`.

The correct interpretation of `id()` is therefore that it provides a **point-in-time unique identifier**. The guarantee is narrow: if you call `id()` on two objects _at the same moment_ and the results are equal, those two names refer to the same object. There is no guarantee that a stored ID value, examined later, still refers to the same object — or refers to any live object at all.

# The Hazard of Storing IDs
This point-in-time limitation means you should **not store IDs** in data structures unless the storage itself keeps the underlying object alive. A tempting but broken pattern is using a set of IDs as a "seen" tracker for non-hashable objects:
```python
seen_ids = set()
for item in collection:
    # item is not hashable, so we can't do: seen_ids.add(item)
    seen_ids.add(id(item))   # appears to work, but is wrong
```

If any item is garbage-collected between when its ID was stored and when a later item is checked, that memory may be reused by a new object, which will then falsely appear to have been "seen before." In a tight loop, CPython reuses memory quickly enough that this can fail almost immediately.

A safer approach is to store the objects themselves in a dictionary keyed by ID:
```python
seen = {}
for item in collection:
    seen[id(item)] = item    # the dict holds a reference, preventing GC
```

The reference held by the dictionary ensures the object remains live for as long as the entry exists, so the ID cannot be recycled while that entry is present.

This problem has a direct analogue in shell scripting. When a shell script spawns a child process, it might capture the child's process ID (PID) with the intention of terminating it later. But PIDs, like CPython memory addresses, are a finite resource. If the child terminates on its own and a new, unrelated process is launched, that new process may receive the same PID. A delayed `kill` targeting the stored PID would then kill the wrong process entirely. The correct solutions — PID namespaces, process groups, or `PR_SET_PDEATHSIG`-style mechanisms — all work by establishing a structural relationship between parent and child that does not depend on PID values remaining stable.

# Equality Is Transient; Identity Is Permanent
These two observations lead to a fundamental asymmetry between equality and identity as **temporal properties**.

**Equality is transient.** Two objects that are equal at one point in a program may not be equal later, even without any name rebinding. One of the objects may be mutated — an element appended, a key deleted — and equality no longer holds. The equality relationship describes a snapshot of the objects' contents at a particular moment.

**Identity is permanent up to name rebinding.** If two names refer to the same object at some point in the code, they will continue to refer to that same object at every subsequent point — unless a name rebinding occurs:
```python
xs = ys = [1, 2, 3]

assert xs is ys

# xs = ...

assert xs is ys
```

Name rebindings are easy to spot: they take the form of `x = ...`, `x := ...`, or direct manipulation of the namespace (e.g., `globals()['x'] = ...`), and there is a small, well-defined set of constructs that can produce them. Mutations, by contrast, can occur through any method call or subscript assignment that the object's type supports, making them far more numerous and harder to track statically.

# No Strict Relationship Between Equality and Identity
It is tempting to assume that one property implies the other, but neither does. Two objects may be equal without being identical (two independently constructed lists with the same contents). Two objects may be both equal and identical (any name and a second name bound to the same object via `zs = xs`). Identity does not imply equality, and equality does not imply identity — there is no strict logical relationship in either direction.

The most striking case where identity does _not_ imply equality is IEEE 754 `NaN`:
```python
nan = float('nan')
nan == nan   # False
nan is nan   # True
```

A `NaN` value is not equal to itself by the IEEE standard, yet it is trivially identical to itself. This is the one case in common use where an object fails to be equal to itself. Since `__eq__` can be implemented arbitrarily — always returning `False`, always returning `True`, or doing anything else — there is nothing in the language that prevents you from building a class with the same behavior, but `float('nan')` is the only instance of this that appears with any regularity in practice.

# Identity Checks Are Faster
Because `is` cannot be overridden, CPython implements it as a direct pointer comparison — it does not call any Python-level `__eq__` method, does not construct a new evaluation frame, and runs no custom code. The `==` operator, on the other hand, must dispatch to `__eq__`, which may do arbitrarily complex work.

One concrete application of this is Python's `enum.Enum`. Enum members are singletons, meaning each member is a unique object that exists only once. Checking membership with `is` rather than `==` exploits this:
```python
from enum import Enum

class Direction(Enum):
    NORTH = 'north'
    SOUTH = 'south'
    EAST  = 'east'
    WEST  = 'west'

# Identity check — no __eq__ dispatch
if x is Direction.NORTH:
    ...

# Equality check — dispatches to __eq__
if x == 'north':
    ...
```

In principle, the `is` comparison should be faster because it skips the equality dispatch entirely. In practice, the difference is extremely difficult to observe in benchmarks for short strings, because Python's string equality check is itself highly optimized — the speedup is buried in system noise. The advantage becomes measurable only with very long strings or under circumstances where the dispatch overhead accumulates significantly.