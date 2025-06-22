---
date: 2025-06-05
modified: 2025-06-22T11:51:42+02:00
---

*This article is entirely based on a [YouTube video by Christopher Okhravi](https://www.youtube.com/watch?v=FdFBYUQCuHQ)*.

---
# Covariance and Contravariance: What They Are and When They're Safe
Many people find this question tricky: Can you use a list of cats where a list of animals is expected? If all cats are animals, shouldn't a list of cats also be a list of animals? Most of the time, the answer is **no**. This topic is about something called **variance**.

# Understanding Variance with Simple Examples
Imagine you ask for a **bag of fruits**, and you get a **bag of apples**. You should be happy! You wanted a bag from which you could take out fruits. And you got a bag of apples. Since apples are a type of fruit, everything you take out of that bag will still be a fruit. So, what you got is something that gives you a _more specific type_ (apples) than what you asked for (fruits). This idea is called **covariance**.

Now, let's look at the opposite idea: **contravariance**. Imagine you ask for a machine that can **juice oranges**. This machine takes oranges and makes juice. But then, you get a machine that can juice **any fruit**. You should also be happy! Since oranges are a type of fruit, you can still put oranges into this machine. But because it's a general machine, you can also put _any other fruit_ into it. In this case, what you got is something that accepts a _more general type_ (any fruit) than what you asked for (oranges). This is called **contravariance**.

# Variance in Code: How Types Relate
Variance describes how the "type relationship" (like when one type is a specific kind of another type) works for things that _use_ other types. It sounds complicated, so let's use a diagram idea.

## Covariance
![](Pasted%20image%2020250605231049.png)

Imagine a `FruitSequence` that can give you `Fruit` objects, and an `AppleSequence` that is a subtype of `FruitSequence`. When you ask the `AppleSequence` for the `next()` method, you get an `Apple`. Since `Apple` is a `Fruit`, this still works perfectly when a `Fruit` is expected.

- `Apple` is a subtype of `Fruit`.
- `AppleSequence` is a subtype of `FruitSequence`.
- The `AppleSequence` (the more specific type) gives you **more specific output** (`Apple` instead of `Fruit`).
- **The arrows go in the same direction**: the contained type (`Apple` vs `Fruit`) and the container type (`AppleSequence` vs `FruitSequence`) both follow the same subtype direction.
## Contravariance
![](Pasted%20image%2020250605231409.png)

Now, consider `OrangeJuicer`, which juices `Orange` objects. And `FruitJuicer`, which is a subtype of `OrangeJuicer`. The `FruitJuicer` has a `juice()` method, but this method takes a `Fruit` as input (a more general type), not just an `Orange`. This works because if it can juice any fruit, it can definitely juice an orange.

- `Orange` is a subtype of `Fruit`.
- `FruitJuicer` is a subtype of `OrangeJuicer`.
- The `FruitJuicer` (the more specific type) accepts **more general input** (`Fruit` instead of `Orange`).
- **The arrows go in opposite directions**: `FruitJuicer` is a subtype of `OrangeJuicer`, but `Orange` is a subtype of `Fruit`. The more general type (`Fruit`) is found in the subtype (`FruitJuicer`).

# Formal Definitions
Let's define these more clearly:
- **Covariance**: If type A is a subtype of type B (A is more specific than B), then a container type that uses **A** (like `List<A>`) is a subtype of a container type that uses **B** (like `List<B>`). The "can be used instead of" rule works the same way for the container and the contained type.
    - Example: An `AppleSequence` (uses `Apple`) can be used where a `FruitSequence` (uses `Fruit`) is needed, because `Apple` is a subtype of `Fruit`.
- **Contravariance**: If type A is a subtype of type B (A is more specific than B), then a container type that uses **B** (like `Juicer<B>`) is a subtype of a container type that uses **A** (like `Juicer<A>`). The "can be used instead of" rule is reversed for the contained type.
    - Example: A `FruitJuicer` (uses `Fruit`) can be used where an `OrangeJuicer` (uses `Orange`) is needed, because `Orange` is a subtype of `Fruit`.
- **Invariance**: This means a type is neither covariant nor contravariant.

# Why Not Always Use Variance? (Type Safety)
You cannot always use covariance and contravariance if you want to keep your code **type-safe** (meaning no unexpected errors related to types during runtime).

Going back to our first question: A `List<T>` (a generic list where T is any type) is usually **invariant** for its type parameter. This means `List<Cat>` cannot be used as `List<Animal>`, and `List<Animal>` cannot be used as `List<Cat>`.

Why? Because a list can do two things:
1. **Take items as input**: You can `add` items to the list.
2. **Give items as output**: You can `read` items from the list.

Here's the rule:
- **Covariance is only safe for output.**
- **Contravariance is only safe for input.**

Since a `List` does _both_ input and output, it can't be safely covariant or contravariant, so it must be invariant.

Another way to remember this rule is the **Robustness Principle**: **"Be conservative in what you do and be liberal in what you accept from others."**
- **Output (Covariance):** If you are a specific type (the subtype), your methods should return items that are either the same type as what the general type (super type) returns, or a _more specific type_. This way, people using your type won't be surprised when they expect a certain kind of item.
- **Input (Contravariance):** If you are a specific type (the subtype), your methods should accept items that are either the same type as what the general type (super type) accepts, or a _more general type_. This way, people using your type won't be surprised when they give you an item that the general type would accept.

# Why This Rule is True for Type Safety
Let's confirm why this rule is needed to avoid errors.

**Problem with Covariance for Input (List of Cats as List of Animals)**. Imagine you could use `List<Cat>` where `List<Animal>` is expected.
- **Reading (Output):** When you take items out, you expect `Animal`s. Since the actual list is `List<Cat>`, you'll get `Cat`s. All `Cat`s are `Animal`s, so this is safe. The output is fine with covariance.
- **Adding (Input):** Since the code expects `List<Animal>`, you should be able to add _any_ `Animal` (like a `Dog` or a `Duck`). But your real object is `List<Cat>`, which cannot hold `Dog`s or `Duck`s. This would cause an error! So, covariance is **not safe** when you add items (input).

**Problem with Contravariance for Output (List of Animals as List of Cats)**. Imagine you could use `List<Animal>`where `List<Cat>` is expected.
- **Adding (Input):** The code expects to put `Cat`s into the list. Since your actual object is `List<Animal>`, you can put _any_ `Animal` into it, which includes `Cat`s. This is safe. Input is fine with contravariance.
- **Reading (Output):** The code expects to get only `Cat`s out of the list. But your actual object is `List<Animal>`, which might contain `Dog`s or `Duck`s. If you try to take a `Dog` out and treat it like a `Cat`, you'll get an error! So, contravariance is **not safe** when you get items (output).

Because of these problems, most programming languages that support variance follow these rules: **contravariance for input and covariance for output.**

# Where Can You See Variance in Code?
Variance commonly appears in three main areas:
1. **Generics (like `List<T>`)**: In languages like C#, you can make interfaces be variant. Special keywords like `in` (for contravariance in input) and `out` (for covariance in output) are used. This matches the input/output rules we just discussed.
2. **Classes**: Many languages support some variance in classes. For example, C# allows "covariant return types" (a method in a specific class can return a more specific type than the method in its general parent class). But C# doesn't allow "contravariant input" directly in classes; for that, you typically need interfaces.
3. **Functions**: In languages where you can pass functions around like regular objects (e.g., using "delegates" in C#), you often find support for variance. Functions are generally **contravariant in their input types** (they can accept more general arguments) and **covariant in their return types** (they can return more specific results).

# What to Remember
Here's a quick summary of what's important about variance:
1. **Covariance**: Allows a type to be replaced by a **more specific type**.
2. **Contravariance**: Allows a type to be replaced by a **more general type**.
3. **Invariance**: Means the type is neither covariant nor contravariant.
4. **Safety Rule**: **Contravariance is safe for inputs**, and **covariance is safe for outputs**.

Understanding these ideas about variance will help you better understand other important programming principles, like the Liskov Substitution Principle.
