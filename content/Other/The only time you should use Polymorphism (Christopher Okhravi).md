---
date: 2025-06-05
modified: 2025-06-05T22:32:33+02:00
---
# When to Use Polymorphism: Behavior Over Data
A crucial principle in software design dictates: **the only appropriate use of subtype polymorphism is when you have variations in behavior in the subtypes, not when you have variations in data.** Understanding this distinction is vital for writing effective and maintainable code.
# The core principle: Replace Conditional with Polymorphism
The best reason to use subtype polymorphism is to avoid using many "if-then" (or `if-else` / `switch`) rules in your code. When you use polymorphism correctly, you don't need these rules. This makes your code **simpler and easier to change**. This idea is often called "**replace conditional with polymorphism**".

But if you use polymorphism wrongly, for things that only have different data, your code will become **more confusing and harder to change**. If your code parts just have different information, a normal class (a blueprint for making objects) is usually all you need. A well-made class can fix problems perfectly when only the data is different. Polymorphism is for when the _ways_ things work, or the _logic_ they follow, are truly different.
# Example 1: Wrong Use – Only Data Changes
Let's look at a simple drawing of code ideas (called a UML diagram):
![](Pasted%20image%2020250605221123.png)

At the top, there's a general type called `IAttack`. Below it are two specific types: `Thunderbolt` and `Scratch`. Think of these as attacks in a game, like in Pokémon. The `IAttack` type says that all attacks must have a `Name` (a word) and `Damage` (a number). `Name` is the attack's title, and `Damage` is how much harm it does:
* For `Thunderbolt`, its `Name` would be "Thunderbolt" and its `Damage` might be 100.
* For `Scratch`, its `Name` would be "Scratch," and its `Damage` maybe 50.

**What's happening here?** This clearly shows **differences in data**. Things like `Name` and `Damage` are about holding information. Even if you can calculate some data, the main idea of these is to store information. The only difference between `Thunderbolt` and `Scratch` here is the _values_ of their names and how much damage they do.

**What's the answer?** For this problem, you **don't need subtype polymorphism**. A single, normal `Attack` class is better. When you make a new `Attack` (like `new Attack()`), you would just give it the name and damage number. The important point is that `Thunderbolt` and `Scratch` should be **objects** (specific things made from the `Attack`class), not their own separate classes, because their differences are just about data.
# Example 2: Still Wrong Use – Data Inside Actions
The next drawing changes a little:
![](Pasted%20image%2020250605221407.png)

`IAttack` is still there, but now it also says attacks must have a `Use` action (a "method"). This `Use` action takes a `player` (the `target`) and doesn't give back any result. The idea is that an attack object would be "used on" a specific player. `Thunderbolt` and `Scratch` are still there, and they both have this `Use`action.

Let's see how they work:
- In `Thunderbolt`'s `Use` action, the code might be `target.Health = target.Health - 50;` (taking away 50 from the player's health).
- In `Scratch`'s `Use` action, it might be `target.Health = target.Health - 25;` (taking away 25 from the player's health).

**What's happening here?** Is this a difference in actions or in data? This is **still a difference in data**. The only thing that changes between the two is the amount being subtracted (50 versus 25). This isn't a deep change in *how* the action is done, but just a difference in the *number* used inside the action.

**What's the answer?** Just like before, if your only differences are in data, **you don't need subtype polymorphism**. A normal class is still the simpler, more flexible, and easier-to-fix choice. You could remove these specific types and just keep one `Attack` class. Its `Use` action would then take a `damage` number. This `damage` number (like 50 or 25) would be used inside the action. The specific damage numbers are data, not truly different actions.
# Example 3: The Right Use – Real Action Differences
The third example finally shows a good way to use polymorphism:
![](Pasted%20image%2020250605221752.png)

The main type is now called `IMove`, which is a broader idea. It still needs a `Use` action that takes a `player` (the `target`) and gives no result back.

However, the specific types are now `Attack` and `Heal`. Both do the `Use` action as `IMove` says, but they do it in very different ways.
- **`Attack`'s `Use` action:** This would be like the earlier examples, reducing the target's health by a certain number (e.g., `target.Health = target.Health - X`, where X is the damage from the specific `Attack` object). `Attack`itself could still be a class, with different attack *objects* representing things like Thunderbolt or Scratch.
- **`Heal`'s `Use` action:** Imagine this one setting the target's health to a full amount, and maybe also setting their armor (e.g., `target.Health = 100;` and `target.Armor = 100;`). Also, while an attack is usually for an *enemy*, a heal action would logically be for *yourself* or a friend.

**What's happening here?** This is where the big difference is that **`Heal` does something completely different from `Attack`**.
- `Attack` takes away health.
- `Heal` _sets_ health to a specific number, and it also changes a totally *different thing* (`Armor`) that `Attack`doesn't even touch.

**The way the action is done is different.** It's not just that the *number* changes; the *kind of action* being performed is different. This is what we mean by "behavioral variation" (differences in actions).

**What's the answer?** You **cannot easily solve this problem with just one normal class without using many "if-then" rules**. If you tried to use one class for both `Attack` and `Heal` without polymorphism, you would need `if-then` rules to decide if you should take away health or set health/armor based on some type. This brings back the confusion and makes the code harder to change, which polymorphism is supposed to fix. So, in this situation, using subtype polymorphism makes the solution **simpler and more flexible**.
