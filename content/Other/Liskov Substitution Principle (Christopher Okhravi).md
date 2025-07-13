---
date: 2025-07-01
modified: 2025-07-01T22:17:04+02:00
---

*This article is entirely based on this [YouTube video by Christopher Okhravi](https://www.youtube.com/watch?v=7hXi0N1oWFU)*.

---

## The Liskov Substitution Principle: Making Code Predictable and Stable
Breaking the Liskov Substitution Principle (LSP) makes our programs hard to understand. This can cause bugs that are difficult to find, and ultimately, make our systems hard to change. Let's explain the **seven rules** that make up this principle.

At its core, LSP says that you should be able to **use an object of a specific type (a subtype) anywhere an object of its more general type (super type) is expected, without causing problems or unexpected behavior in the program.**

You might think that computer compilers (which turn your code into a program) already handle this. And yes, they help! But it's very possible, and quite easy, to write code that breaks LSP even if the compiler doesn't complain.

The Liskov Substitution Principle has **three rules about types** and **four rules about behavior**.
- The three type rules are usually checked by compilers.
- But the four behavioral rules are up to us, the programmers, to follow. If we break these behavioral rules, we violate LSP, and our systems become very hard to understand.

So, in statically typed languages (which check types before running) like C# or Java you can't really break the type rules. But you can definitely break the behavioral rules.

Let's understand LSP by looking at its **formal definition**, then a helpful rule called the **Robustness Principle**, and finally, the specific **type** and **behavioral rules**.

# The Formal Definition
The formal definition sounds complex, but is simple:

> "Let `P(x)` be a property provable about objects `x` of type `T`. Then `P(x)` should be true for objects `y` of type `S` where `S` is a subtype of `T`.

In plain words: **Whatever is true about objects of the super type must also be true about objects of the subtype.** This means the **subtype must respect the "contract" of its super type**. Objects of the subtype must act in a way that makes sense, matching how we expect objects of the super type to act.

This might seem simple, but defining "*not behaving unexpectedly*" is tricky. That's why we have these seven rules.

# The Robustness Principle (Postel's Law)
A simple way to remember the main idea of these rules is the **Robustness Principle**, also known as Postel's Law: **"Be conservative in what you do, but be liberal in what you accept from others."**

This means: we should be careful not to make mistakes in our own code, but we should be forgiving of mistakes (or different inputs) from other parts of the system.

The Robustness Principle basically tells us we should use **contravariance for input** and **covariance for output**:
![](Pasted%20image%2020250701221100.png)

- **For Input (Contravariance - Be Liberal)**: If a specific type (subtype) has an action (method) that is also in its general type (super type), this action should be "liberal" in what it takes. It should accept whatever the general type accepts, or **even more general** things.
    - Imagine a set of values that the general type's method can take. The specific type's method must be able to accept that same set of values, or a _bigger set_ (a superset). This is contravariance.
- **For Output (Covariance - Be Conservative)**: If a specific type (subtype) is good to use with the Robustness Principle, then its action should be "conservative" in what it gives back. It should only return values that are part of what the general type's method can return, or **something smaller** (a **subset**).
    - The possible values that the specific type's method can return must be a subset of the values the general type's method can return. This is covariance.

The ideas of "covariance" and "contravariance" are the keys to understanding the LSP rules.

# The Three Type Rules
To follow the Liskov Substitution Principle, we must follow these three type rules. Compilers usually check these for us:
1. **Method Parameters Must Be Contravariant**: If an action (method) is in both the specific type (subtype) and the general type (super type), the types of its inputs in the specific type must be the *same as* in the general type, or *more general* types. This is because the specific type must be "liberal in what it accepts."
2. **Method Return Types Must Be Covariant**: If an action (method) is in both types, the type of what it gives back in the specific type must be the *same as* in the general type, or a *more specific type*. This is because the specific type must be "conservative in what it does" (sends out). It needs to return values from the same group or a smaller group.
3. **No New Exceptions**: The specific type must not throw any new types of errors (exceptions) that the general type does not. Any errors it throws must be the _same type_ as errors from the general type, or _more specific types_ of those errors. If a specific type suddenly throws a completely new error, anyone using it as if it were the general type would be surprised, breaking the contract.

These Type Tules are often automatically enforced by programming language compilers, though not all languages check all rules.

# The Four Behavioral Rules
These rules are about how the code _acts_ and are not usually checked by compilers. We, as developers, must make sure our code follows them:
1. **Preconditions Same or Weaker**: Preconditions define what must be true _before_ a method can be called. In the specific type (subtype), these rules must be the _same_ or _less strict_ (weaker) than in the general type (super type). This aligns with being "liberal in what you accept." The specific type should not ask for _more_ than the general type does.
2. **Postconditions Same or Stronger**: Postconditions define what must be true _after_ a method has finished its work. In the specific type (subtype), these rules must be the _same_ or _more strict_ (stronger) than in the general type (super type). This aligns with being "conservative in what you do." The specific type should guarantee at least as much as the general type, or even more.
3. **Single-State Invariants Same or Stronger**: An invariant is something that must always be true about an object. For things that are true about an object at any single moment (single-state), the rules in the specific type (subtype) must be the _same_ or _more strict_ (stronger) than in the general type (super type). If something must always be true in the general type, it must also be true in the specific type. Making it even stronger in the specific type is fine, because a stronger rule still meets the weaker rule.
4. **Multi-State Invariants Same or Stronger**: This is similar to single-state invariants but applies to conditions that must remain true over multiple changes or states of an object. The rules in the specific type must be the _same_ or _more strict_ (stronger) than in the general type. For example, if a general type says a number must always increase, a specific type could say it must always increase by at least two – this is a stronger, but still valid, rule.

#### Understanding Pre/Postconditions and Invariants

- **Precondition**: What must be true _before_ an action. Example: A method needs a positive number. If the general type requires a "positive number," the specific type can allow "any number greater than or equal to zero" (weaker precondition). It cannot require "only numbers greater than 10" (stronger precondition).
    
- **Postcondition**: What must be true _after_ an action. Example: A method that gets the absolute value of a number must return a non-negative number. If the general type says the list length will be zero after `clear`, the specific type must also guarantee this, or something even stricter (like also guaranteeing the memory is released). It cannot say the length _might_ be something else (weaker postcondition).
    
- **Invariant (Single-State)**: What must always be true about an object at any given moment. Example: A list that is never empty (always has at least one item). If the general type says length >= 1, the specific type can say length >= 2 (stronger). It cannot say length >= 0 (weaker), because then it could be empty, breaking the parent's rule.
    
- **Invariant (Multi-State)**: What must be true about an object across different changes or states. Example: A list where its size always increases. If the general type says length always increases, the specific type could say it always increases by two (stronger). It cannot say the length "increases or stays the same" (weaker), because that would break the parent's rule.
    

**Why these behavioral rules are critical:** If a specific type makes preconditions stronger or postconditions weaker, or weakens its invariants, then an object of that specific type **cannot safely be used** where an object of the general type is expected. This breaks the whole idea of LSP and leads to surprising, incorrect behavior.

### Conclusion

The Liskov Substitution Principle has three type rules (checked by compilers) and four behavioral rules (our responsibility as developers). LSP is not just a theory; it has real effects on our code.

Breaking LSP leads to code that is hard to understand and acts in unexpected ways. This can cause bugs that are very hard to find, making our systems difficult to change. In business, this means we can't keep up with changes and grow.

So, next time you create a new specific type (subtype), pause and ask yourself if it truly respects the "contract" of its general type (parent). Everyone who uses your code will thank you!

---

Does this version make LSP clearer for you? Are you ready to move on to the next video?
