---
date: 2025-06-05
modified: 2025-06-22T11:51:21+02:00
---

*This article is entirely based on a [YouTube video by Christopher Okhravi](https://www.youtube.com/watch?v=TkUhAbbRCAE)*.

---

A classic idea in programming is: **couple to abstractions (general ideas) and not to concretions (specific things)**. But how do we actually do this? In the book "Elegant Objects" by Yegor Bugayenko, there's a chapter called "Always Use Interfaces." It talks about why it's important to connect to general ideas because it makes code easier to change.

Here's what the book suggests: **"Make sure that all public methods in your class implement some interface."**

This means if you have a public method in your code that doesn't come from a general "interface" above it, then this method is making other parts of the code connect directly to this specific piece of code (**I'm encouraging to couple to concretions**). This is bad because we **should couple to general ideas (interfaces)** instead. If there was a general "**interface**" above your specific code, other parts of the code could connect to that interface. This makes it less likely that they'll connect to something too specific. It's a simple but powerful idea.
# Connecting to Past Ideas: Polymorphism and Data vs. Behavior
In an earlier discussion, it was explained that you should only use different types (subtypes) when they have different **actions or behaviors**, not just different **data**. If you use subtypes for data differences, your code becomes harder to change.

Let's look at an [example we discussed before](The%20only%20time%20you%20should%20use%20Polymorphism%20(Christopher%20Okhravi).md). We had an `IAttack` interface with a `Use` method that takes a `target` (a player). Imagine this is for a game where players fight using different attacks. Then, we had two specific types: `Thunderbolt` and `Scratch`. These were classes that used the `IAttack` interface:
![](Pasted%20image%2020250605224843.png)

However, the problem we found was that the only difference between `Thunderbolt` and `Scratch` was how much damage they did (50 or 25). So, the only difference was in their **data**, not in their **actions** or how they worked. This means it wasn't a good way to use subtypes.

Instead, those data differences should be handled by making different **objects** from a single class, giving them different numbers (parameters) when you create them. So, instead of two specific types, we'd have just one `Attack` class. Then, you'd make objects of this `Attack` class, and each object could represent what used to be `Thunderbolt` or `Scratch`. By doing this, we fix the problem of using subtypes for just data differences. Now, we use classes for data differences, which is good.
![](Pasted%20image%2020250605225013.png)
# The Missing Piece: Abstractions and Interfaces
But what does all this have to do with connecting to general ideas (abstractions) instead of specific things (concretions)?

When we changed our code to use a single `Attack` class, we lost our original general idea: `IAttack`. Having `IAttack` as a general idea is good because **it makes other parts of the code connect to this general idea** instead of the specific `Attack` class (the **concretion**).

But now, since we removed `IAttack`, we only have the specific `Attack` class. This breaks the rule from "Elegant Objects" about "Always Use Interfaces." Now, we are making other parts of the code connect to this specific class, which is not good.

So, when we say that using subtypes for data differences isn't good, it doesn't mean we should connect to specific classes (concretions). **We should still connect to general ideas (abstractions).** It only means that if your only differences are in data, you shouldn't make separate subtypes for those data differences. But that doesn't mean your class can't have a general idea (an interface) above it. If you follow the "Always Use Interfaces" rule, it should.

This is why, in the previous discussion, we introduced a higher-level general idea called `IMove`. Our `Attack` class then became one specific type of `IMove`. Is `Attack` the only type of `IMove`? Probably not. We can easily imagine many other "moves" in a game.

By choosing this plan, we get the best of both worlds. We use **classes for data differences** (all attacks are handled by the `Attack` class). But we still **encourage coupling to abstractions** because we provide a general idea (`IMove`) above our `Attack` class. Even if we don't have other types of `IMove` right now, it's easy to see what another one might look like. If it's clear what another subtype could be, you shouldn't worry about making a general idea too early. Adding this interface is usually easy, and then you can connect to this interface. This prepares you to add more specific types later. However, if it's not clear what another subtype might look like, then it's better to stop and think more about what the general idea truly is.
![](Pasted%20image%2020250605225300.png)
# Why Connect to General Ideas? The Battle Menu Example
We've talked about how tempting it is for other parts of the code to connect to specific classes instead of general ideas. What might this "other class" be?

Imagine a turn-based game with two players. You might have a **battle menu**. This menu shows you different moves you can make. It's clear that not all moves have to be attacks. Some moves could be magic or other special actions in your game.

If this battle menu connects directly to the specific `Attack` class, then it's tightly tied to attacks. When you realize you want more than just attacks (like magic), you'll have to change the battle menu.

But if, instead of connecting to the specific `Attack` class, the battle menu connects to the general idea, the `IMove`interface, then you are ready. Even if you only use attacks at first, you're prepared for when you inevitably decide you want more than just attacks. Because the other possible types are clear, you get the benefits of this design. The battle menu would connect to a general idea, not a specific one.

So, you can see that these two ideas are not in conflict. The point about subtypes was simply that you shouldn't use them to handle only data differences. But that doesn't mean you have to connect to specific classes. You can and should still connect to general ideas.
# In Summary
- We should avoid making subtypes that only differ in their **data**, because this makes code harder to change.
- Instead, we should use **classes** to handle differences in data, so different objects can hold different information.
- But we should **not** make it easy for other parts of the code to connect to specific classes.
- We should use classes (not subtypes) for data differences, but also make sure that all public actions (methods) in our specific classes are actually part of a general idea (an interface) above them. This encourages others to connect to general ideas, not specific ones.