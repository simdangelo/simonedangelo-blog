---
date: 2026-06-23
modified: 2026-06-23T15:39:05+02:00
---
# Inheritance as a Live View Mechanism
The live-view/snapshot distinction extends naturally into object-oriented design. **Inheritance**, it turns out, is a form of live view: **a derived class always reflects the current state of its base class, including changes made after the derived class was defined**.
```python
class Base:
    def f(self):
        return 'f'

class Derived(Base):
    pass

# After Derived is defined, add a method to Base at runtime
Base.g = lambda _: ...

d = Derived()
d.g()   # works — Derived has a live view of Base
```

This is rarely done at runtime deliberately, but the same mechanism operates across software versions. Suppose `Base` ships with method `f` in version 1, gains method `g` statically in version 2, and acquires method `h` at runtime during execution. The behavior of each design approach differs significantly:

- **Inheritance**: `Derived` automatically has `f`, `g`, and `h` — all changes to `Base` are immediately available, with no work required on the `Derived` side.
- **Composition**: A class that holds a `Base` instance and delegates explicitly must manually expose each method it wants to surface. It will have `f` from the initial implementation, but `g` and `h` are not available unless deliberately added.
- **Object construction** (copying methods explicitly at class-creation time): offers fine-grained control — you can selectively include `f` and `g` while excluding `h`, or any other combination you choose.

This framing reframes the classic composition-versus-inheritance debate in concrete terms. If `Base` is something like a `networkx.DiGraph` — a large, actively developed class with many methods that change across library versions — subclassing it means you automatically benefit from every new method without tracking the changelog. You implement one focused piece of behavior and remain transparent to the rest. That is precisely the scenario where inheritance is the right tool.

Composition, by contrast, gives you control over exactly which behaviors you expose, but that control comes with a corresponding responsibility: every new method in the base that you want to surface must be explicitly added to your wrapper. If you are building something where you need precise authority over the interface — and where you are willing to maintain that interface carefully — composition is the better choice.

## `collections.ChainMap` as a Snapshot/Live-View Mechanism

The snapshot and live-view distinction can also be implemented explicitly using `collections.ChainMap`, which provides a layered view over multiple dictionaries without merging them:

```python
from collections import ChainMap

base = {'x': 1, 'y': 2}
snapshot = {**base}         # shallow copy — changes to base are not visible
live = ChainMap({}, base)   # layered view — changes to base are visible

base['z'] = 3
print('z' in snapshot)      # False — snapshot does not reflect the change
print('z' in live)          # True  — live view reflects the change
```

The `ChainMap` always reads through to the base layer for keys not present in the outer layer. However, writes to the `ChainMap` go into the outermost dictionary only — they do not propagate back down to `base`. This creates an asymmetry: changes to `base` are visible through `live`, but changes made through `live` shadow `base` without modifying it.

That shadowing behavior is important to understand, because it is also a source of subtle bugs.

## Shadowing: When a Write Does Not Go Where You Expect

**Shadowing** occurs when a name or key at one layer of a multi-layered structure blocks visibility of the same name or key in a lower layer. The ChainMap is a direct example, but the same phenomenon appears throughout Python in several forms.

### Rebinding vs. Mutating

Consider the difference between rebinding a name and mutating the object it refers to:

```python
xs = [1, 2, 3]
ys = xs             # ys is a live view of xs

# Option 1: rebind xs to a new empty list
xs = []             # xs now refers to a new object; ys is unaffected

# Option 2: mutate xs in place
xs.clear()          # the shared object is cleared; ys reflects the change

# Option 3: mutate via slice deletion
del xs[:]           # equivalent to xs.clear(); ys reflects the change
```

Assigning `xs = []` creates a new list and rebinds the name `xs` to it. The original list — which `ys` still refers to — is untouched. Calling `xs.clear()` or `del xs[:]` mutates the original object in place, so the change is visible through both `xs` and `ys`. Which behavior you want determines which form to use, and confusing them is a common source of bugs.

### `sys.path` and the Shadowing Trap

This distinction has a direct practical consequence for `sys.path` manipulation. There are several seemingly equivalent ways to prepend a directory to the import path, but they do not all work:

```python
import sys
from sys import path

# Works — mutates the actual sys.path list in place
sys.path.insert(0, '/my/dir')
sys.path.append('/my/dir')

# Does NOT work — creates a local shadow, does not affect sys.path
path.insert(0, '/my/dir')  # works, path is still the same object
path = ['/my/dir'] + path  # creates a new list bound to local 'path'; sys.path unchanged

# Works — reassigns back into the sys module's namespace
sys.path = ['/my/dir'] + sys.path
```

The form `from sys import path` binds the local name `path` to the same list object as `sys.path`. Calling `path.insert()` or `path.append()` mutates that shared list, so it works. But `path = ['/my/dir'] + path` creates a brand-new list and rebinds the local name `path` to it — `sys.path` inside the `sys` module is never touched, and the import path is not changed. The local `path` variable has shadowed `sys.path` without affecting it.

### Shadowing in the Inheritance Hierarchy

The same dynamic appears in class attribute lookups through the MRO (method resolution order). When Python evaluates `obj.x`, it walks the MRO — first the instance's own `__dict__`, then the class, then base classes in order — and returns the first match it finds. A value at a higher level of the hierarchy shadows any value with the same name at a lower level.

```python
class Base:
    x = 10

class Derived(Base):
    pass

d = Derived()

# Mutating assignment on Base — visible everywhere
Base.x = 20
print(Base.x)     # 20
print(Derived.x)  # 20 — no shadow, MRO finds Base.x

# Mutating assignment on Derived — creates a shadow
Derived.x = 99
print(Derived.x)  # 99 — found in Derived.__dict__
print(Base.x)     # 20 — Base is unaffected

# Now Base.x = 30 is invisible through Derived
Base.x = 30
print(Derived.x)  # still 99 — shadow blocks the update from Base
```

Assigning to `Base.x` changes the value that is found when the MRO searches `Base`, so both `Base.x` and `Derived.x` (which falls through to `Base`) reflect the change. Assigning to `Derived.x` inserts a new entry into `Derived.__dict__`, creating a shadow. From that point forward, any lookup of `Derived.x` finds the shadowed value first and never reaches `Base.x`.

This matters particularly when the shared attribute is **immutable**. Because immutable objects cannot be updated in place, every change must go through assignment — and assignment always writes to the level of the hierarchy you specify. If `Derived.x = value` was executed at some point, a subsequent `Base.x = new_value` will not be visible through `Derived`, because the shadow is still in place. To restore visibility, the shadow must be explicitly removed:

```python
del Derived.x      # removes the shadow; Derived.x now falls through to Base again
```

If this is not done, the ordering of assignments matters: performing `Derived.x = ...` before `Base.x = ...` leaves a shadow that blocks the base update, while the reverse order does not — but only until the next assignment to `Derived.x` creates a new shadow.

## Why This Matters

Shadowing errors produce exactly the class of bug that makes debugging Python difficult: either a change you made is visible somewhere you did not intend (spooky action at a distance), or a change you made is invisible somewhere you fully expected it to appear. Both failure modes are consequences of the same underlying mechanism — the difference between mutating an object in place and rebinding a name to a new object, compounded by the layered lookup semantics of `ChainMap`, module namespaces, and the MRO.

Understanding the live-view/snapshot distinction, and recognizing how shadowing interacts with it, does not just help you design cleaner code. More immediately, it gives you a precise vocabulary for diagnosing this family of bugs when they appear.

---

_Next: Hashability versus immutability._
