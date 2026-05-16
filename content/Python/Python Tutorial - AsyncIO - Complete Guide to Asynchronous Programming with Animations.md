---
modified: 2026-05-09T15:36:19+02:00
---

*Content summarized from [YouTube video](https://www.youtube.com/watch?v=oAkLSJNr5zY&t=5502s) by Corey Schafer.*

---

**AsyncIO** is Python's built-in library for writing concurrent code using the `async`/`await` syntax. This guide uses the latest version of Python and focuses on modern approaches, as Async/IO has evolved considerably over the years with different methods for running the event loop, scheduling tasks, and executing concurrent operations.

# What is Concurrency?
With **synchronous code** execution, one operation happens after another. This resembles a Subway restaurant where an employee makes your entire sandwich from start to finish before moving to the next customer. With **concurrent code**, the model resembles a McDonald's where someone takes your order and moves to the next customer while your food is prepared in the background.

Asynchronous execution **doesn't automatically mean faster execution**. It means you can perform other useful work instead of sitting idle while waiting for external operations like network requests, database queries, and similar IO operations. This is why Async/IO excels at **IO-bound tasks**—anytime your program waits for something external.

Async/IO is **single-threaded** and runs on a **single process**. It uses **cooperative multitasking** where tasks voluntarily yield control. For CPU-bound tasks requiring heavy computation, you would use processes instead.

# Core Terminology and Basic Concepts
## Coroutines and Coroutine Functions
A **coroutine function** is defined using the `async def` keywords. When you call a coroutine function, it doesn't immediately execute the function body—it returns a coroutine object. Consider this example:
```python
async def async_function(delay):
    await asyncio.sleep(delay)
    return "async result test"
```

This is a coroutine function. The **coroutine function** is what you define with `async def`, while the **coroutine object** is the **awaitable** returned when you call that function. Coroutines resemble generators in that they can suspend execution and resume later, but they're designed specifically to work with an event loop. They have extra features that Async/IO needs to schedule them, await IO operations, and coordinate multiple tasks.

When you execute `task1 = async_function(1)`, you're not running the function—you're creating a coroutine object. To actually run this coroutine and get the result, you must await it: `result = await task1`.
## The Event Loop
The **event loop** is the **engine that runs and manages asynchronous functions**. It tracks all tasks, and when a task suspends because it's waiting for something external, control returns to the event loop, which finds another task to either start or resume.

You must have a running event loop for any asynchronous code to work. The `asyncio.run()` function handles this:
```python
asyncio.run(main())
```

This function gets the event loop running, executes tasks until they're marked as complete, and closes down the event loop when finished. You cannot call an asynchronous function directly—you need the event loop to manage it.
## The Await Keyword and Awaitables
**Awaitables** are objects that implement a special `__await__()` method under the hood. You use the `await` keyword extensively in asynchronous code. An object must be awaitable for you to use `await` on it.

Synchronous functions like `time.sleep()` or **regular synchronous functions cannot be awaited**. Synchronous libraries don't have a mechanism to work with the event loop—they don't know how to yield control and resume later. They lack the underlying `__await__()` method needed to pause execution and restart later. These capabilities must be explicitly coded into libraries to make them compatible with Async/IO. This is why you cannot await `time.sleep()` and must use `asyncio.sleep()` instead.

To use the `await` keyword, you must be within a function marked with `async`. Attempting to use `await` outside an async function produces an error:
![[Pasted image 20260324231614.png]]

> When you **await** something, you're **telling the event loop to pause execution of the current function and yield control back to the event loop, which can then run another task**. The **current function stays suspended until the awaitable completes**.
## Three Types of Awaitable Objects
Python's Async/IO defines three main types of awaitable objects.
### 1. Coroutines
**Coroutines** are created when you call an async function.

**Coroutines are objects representing computations that can be paused and resumed**. There are actually two distinct concepts: the **coroutine function** (defined with `async def`) and the **coroutine object** (the awaitable returned when calling that function):
```python
import asyncio

async def async_function(test_param: str) -> str:
	print("This is an asynchronous coroutine function.")
	await asyncio.sleep(0.1)
	return f"Async Result: {test_param}"

async def main():
	coroutine_obj = async_function("Test")
	print(coroutine_obj)
	 
	coroutine_result = await coroutine_obj
	print(coroutine_result)


if __name__ == '__main__':
	asyncio.run(main())
```

Here's the result:
```
<coroutine object async_function at 0x101c17c40>
This is an asynchronous coroutine function.
Async Result: Test
```

Coroutines are a bit like *generators*, in the sense that they can suspend execution and resume later, but they're designed to work with an event loop.

When you **call an async function**, it **returns a coroutine object** without executing the function body. To **execute it**, you must **await that coroutine object** with `await`.

When you await a coroutine object directly (as in our example), it is scheduled and immediately awaited, so the caller pauses until it completes. This provides no concurrency benefit because you're scheduling and completing one operation before moving to the next.
### 2. Tasks
**Tasks** are wrappers around coroutines that can be executed independently. Tasks are how you actually run coroutines concurrently. When you wrap a coroutine in a task using `asyncio.create_task()`, it's handed over to the event loop and scheduled to run when the loop gets a chance.

```python
task1 = asyncio.create_task(async_function(1))
```

The task tracks whether the coroutine finished successfully, raised an error, or was cancelled—just like a future would. In fact, tasks are futures under the hood, but with extra logic to actually run the coroutine and perform the intended work. This is why you work with tasks instead of futures in most code.

Unlike coroutine objects, tasks can be scheduled on the event loop without being run immediately. They just sit there until the loop gets control. **This is the key to Async/IO: you can queue up multiple tasks at once, and the event loop will run them whenever it's ready, letting them take turns while waiting on IO.**

When you print a newly created task, it shows as pending:
```python
import asyncio

async def async_function(test_param: str) -> str:
	print("This is an asynchronous coroutine function.")
	await asyncio.sleep(3)
	return f"Async Result: {test_param}"

async def main():
	task = asyncio.create_task(async_function("Test"))
	print(task)
	task_result = await task
	print(task_result)


if __name__ == "__main__":
	asyncio.run(main())
```

Here's the result:
```
<Task pending name='Task-2' coro=<async_function() running at /Users/simonedangelo/dev/fastapi_tutorial/asyncio_test.py:3>>
This is an asynchronous coroutine function.
Async Result: Test
```

The task shows its status (pending), its name, and the coroutine it's wrapping.
### 3. Futures
**Futures** are low-level objects representing eventual results. If you're familiar with JavaScript, futures are similar to promises—they represent a promise of a result that will be available later. However, unlike JavaScript, in Python you almost never work with futures directly.

A future's job is to hold a certain state and result. The state can be:
- **Pending**: The future doesn't have any result or exception yet
- **Cancelled**: The future was cancelled using `future.cancel()`
- **Finished**: The future has either a result (set with `future.set_result()`) or an exception (set with `future.set_exception()`)

Here's what working directly with futures looks like (though you rarely do this):
```python
future = asyncio.Future()
print(future)  # Shows: <Future pending>
future.set_result("future result test")
result = await future
print(result)  # Shows: "future result test"
```

You would only use futures directly if you were writing low-level Async/IO code, such as building an async-compatible framework. In normal application code, you write coroutines, and when you schedule them as tasks, Async/IO uses futures under the hood to track results.

# How Async/IO Works: Detailed Execution Flow
## Example 1: Synchronous Baseline
Consider this synchronous code with no event loop:
```python
import time

def fetch_data(param):
    print(f"Do something with {param}")
    time.sleep(param)
    print(f"Done with {param}")
    return f"Result of {param}"

def main():
    result1 = fetch_data(1)
    print("Fetch 1 fully complete")
    result2 = fetch_data(2)
    print("Fetch 2 fully complete")
    return [result1, result2]


t1 = time.perf_counter()

results = main()
print(results)

t2 = time.perf_counter()
print(f"Finished in {t2 - t1:.2f} seconds")
```

Execution flows like this:
1. Call `main()`
2. Call `fetch_data(1)`
3. Print "Do something with 1"
4. Sleep for 1 second (blocking)
5. Print "Done with 1"
6. Return "Result of 1"
7. Print "Fetch 1 fully complete"
8. Call `fetch_data(2)`
9. Print "Do something with 2"
10. Sleep for 2 seconds (blocking)
11. Print "Done with 2"
12. Return "Result of 2"
13. Print "Fetch 2 fully complete"
14. Return both results
15. Print the results

Total execution time: 3 seconds (1 + 2). Each operation completes entirely before the next begins. Here's the result:
```
Do something with 1
Done with 1
Fetch 1 fully complete
Do something with 2
Done with 2
Fetch 2 fully complete
['Result of 1', 'Result of 2']
Finished in 3.01 seconds
```
## Example 2: Converting to Async (Common Mistake)
A common first attempt converts functions to coroutines but gains no concurrency:
```python
import asyncio

async def fetch_data(param):
    print(f"Doing something with {param}")
    await asyncio.sleep(param)
    print(f"Done with {param}")
    return f"async result of {param}"

async def main():
    task1 = fetch_data(1)  # Creates coroutine object
    task2 = fetch_data(2)  # Creates coroutine object
    
    result1 = await task1  # Schedules AND runs to completion
    print("Task 1 fully complete")
    result2 = await task2  # Schedules AND runs to completion
    print("Task 2 fully complete")
    
    return [result1, result2]


t1 = time.perf_counter()

results = asyncio.run(main())
print(results)

t2 = time.perf_counter()
print(f"Finished in {t2 - t1:.2f} seconds")
```

Here's the result:
```
Doing something with 1
Done with 1
Task 1 fully complete
Doing something with 2
Done with 2
Task 2 fully complete
['async result of 1', 'async result of 2']
Finished in 3.00 seconds
```

Total time: 3 seconds. **No concurrency benefit**.

> **At any given moment, only one task is scheduled and running on the event loop**.

The second task doesn't even get scheduled until the first is completely finished. Let's see what it means.

When `asyncio.run(main())` executes, it **creates the event loop and runs the `main` coroutine**:
![[Pasted image 20260330234028.png]]

Inside the `main` coroutine:
1. `task1 = fetch_data(1)` **creates a coroutine object**—**it does NOT schedule anything on the event loop**; **it just returns a coroutine object**.
2. `task2 = fetch_data(2)` creates another coroutine object. Still nothing scheduled.
3. `result1 = await task1` is where things happen. This line both **schedules the coroutine on the event loop** **AND runs it to completion** at the same time.

Here's the detailed execution flow:
- The `await task1` **suspends the main coroutine**:
	![[Pasted image 20260330234310.png]]
- The **event loop looks for ready tasks**. It finds the `fetch_data(1)` coroutine that was just scheduled:
	![[Pasted image 20260330234443.png]]
- The event loop enters that coroutine and runs until hitting `await asyncio.sleep(1)`:
	![[Pasted image 20260330234520.png]]
- The `await asyncio.sleep(1)` suspends the `fetch_data(1)` task and kicks off a background timer:
	![[Pasted image 20260330234547.png]]
- The `fetch_data(1)` task stays suspended until the timer completes. After 1 second, the timer completes and wakes up the `fetch_data(1)` task:
	![[Pasted image 20260330234609.png]]
- The event loop sees this task is ready and resumes it. It continues from the await, prints "Done with 1", and returns:
	![[Pasted image 20260330234845.png]]
- This completes the task, which **wakes up the main coroutine** (because main was awaiting it). Main continues and prints "Task 1 fully complete":
	![[Pasted image 20260330234957.png]]
- Now `await task2` does the same thing—schedules and runs to completion
- The second task suspends on its sleep for 2 seconds
- After 2 seconds, it wakes, completes, and returns
- Main continues, prints "Task 2 fully complete", and returns both results:
	![[Pasted image 20260330235117.png]]
## Example 3: Correct Async Implementation with Task Scheduling
The correct approach schedules tasks before awaiting them:
```python
import asyncio

async def async_function(delay):
    print(f"Doing something with {delay}")
    await asyncio.sleep(delay)
    print(f"Done with {delay}")
    return f"Result of {delay}"

async def main():
    task1 = asyncio.create_task(async_function(1))
    task2 = asyncio.create_task(async_function(2))
    
    result1 = await task1
    print("Task 1 fully complete")
    result2 = await task2
    print("Task 2 fully complete")
    
    return [result1, result2]

t1 = time.perf_counter()

results = asyncio.run(main())
print(results)

t2 = time.perf_counter()
print(f"Finished in {t2 - t1:.2f} seconds")
```

Here's the result:
```
Doing something with 1
Doing something with 2
Done with 1
Task 1 fully complete
Done with 2
Task 2 fully complete
['Result of 1', 'Result of 2']
Finished in 2.00 seconds
```

Now the execution flow changes significantly:
1. `asyncio.run(main())` creates the event loop and schedules the main coroutine and the event loop starts running the main coroutine:
	![[Pasted image 20260401222122.png]]
2. `task1 = asyncio.create_task(async_function(1))` creates a task and **schedules it on the event loop**. The task is now ready and waiting:
	![[Pasted image 20260401222213.png]]
3. `task2 = asyncio.create_task(async_function(2))` creates another task and **schedules it on the event loop**. Both tasks are now scheduled and ready:
	![[Pasted image 20260401222239.png]]
4. `result1 = await task1` suspends the main coroutine and yields control to the event loop:
	![[Pasted image 20260401222302.png]]
5. The event loop uses a FIFO (First In, First Out) queue for ready tasks. It finds `task1` (`fetch_data(1)`) is ready:
	![[Pasted image 20260401223209.png]]
6. It enters that task and runs until hitting `await asyncio.sleep(1)`:
	![[Pasted image 20260401223246.png]]
7. The await suspends `task1` and kicks off a background timer for 1 second:
	![[Pasted image 20260401223309.png]]
8. `task1` is now suspended, and control returns to the event loop. The event loop looks for ready tasks and it finds `task2` (`fetch_data(2)`) is ready:
	![[Pasted image 20260401223437.png]]
9. It enters that task and runs until hitting `await asyncio.sleep(2)`. The await suspends `task2` and kicks off a background timer for 2 seconds:
	![[Pasted image 20260401223532.png]]

**This is the concurrency point**: Both timers are now running in the background simultaneously. Both tasks are suspended, waiting for their respective timers.

10. After 1 second, the first timer completes:
	![[Pasted image 20260401223652.png]]
11. This wakes up `task1`, marking it as ready:
	![[Pasted image 20260401223710.png]]
12. The event loop finds `task1` is ready and it resumes `task1` where it left off (after the await). `task1` prints `Done with 1` and returns `Result of 1`:
	![[Pasted image 20260401224114.png]]
13. `task1` is now complete
14. This wakes up the main coroutine because main was awaiting `task1`. Main resumes and prints "Task 1 fully complete":
	![[Pasted image 20260401224201.png]]
15. Main reaches `await task2`. `task2` is still suspended (its 2-second timer hasn't finished)
16. Main suspends again, waiting for `task2`:
	![[Pasted image 20260401224306.png]]
17. After another second (2 seconds total), the second timer completes:
	![[Pasted image 20260401224325.png]]
18. This wakes up `task2` and the event loop finds `task2` is ready:
	![[Pasted image 20260401224354.png]]
19. It resumes `task2`, which prints `Done with 2` and returns `Result of 2`
20. `task2` completes, waking up the main coroutine:
	![[Pasted image 20260401224434.png]]
21. Main resumes, prints "Task 2 fully complete", and returns the results:
	![[Pasted image 20260401224452.png]]
	![[Pasted image 20260401224547.png]]
22. The even loop closes down and results will be printed out:
	![[Pasted image 20260401224750.png]]

Total execution time: 2 seconds—the duration of the longest task. **Both tasks ran concurrently**, with their sleep operations happening simultaneously in the background.

Notice that "Doing something with 2" prints before "Done with 1". This confirms both tasks started before either completed.
## Example 4: Understanding Await Order and Execution
An important detail: awaiting a task doesn't guarantee that task runs immediately. The event **loop runs whatever is ready** based on its internal **FIFO queue**. What awaiting guarantees is that execution won't proceed past the await until that awaited operation completes (=*Ciò che garantisce l'uso di `await` è che l'esecuzione non proseguirà oltre il punto in cui è stato utilizzato `await` finché l'operazione in attesa non sarà completata.*).

Consider this variation:
```python
async def main():
    task1 = asyncio.create_task(async_function(1))
    task2 = asyncio.create_task(async_function(2))
    
    result2 = await task2  # Await task2 FIRST
    print("Task 2 fully complete")
    result1 = await task1  # Await task1 SECOND
    print("Task 1 fully complete")
    
    return [result1, result2]
```

Here's the result:
```
Doing something with 1
Doing something with 2
Done with 1
Done with 2
Task 2 fully complete
Task 1 fully complete
['async result of 1', 'async result of 2']
Finished in 2.00 seconds
```

Both tasks are still scheduled in the same order (task1 then task2). When you await task2 first, here's what happens:
1. `await task2` suspends the main coroutine (until task2 is done)
2. The event loop looks for ready tasks
	![[Pasted image 20260401233119.png]]
3. It finds task1 first (because it was scheduled first in the FIFO queue):
	![[Pasted image 20260401233154.png]]
4. Task1 runs until its await, then suspends and kicks off the background sleep:
	![[Pasted image 20260401233253.png]]
5. The event loop finds task2 ready. Task2 runs until its await, then suspends:
	![[Pasted image 20260401233340.png]]
6. Both timers run concurrently. After 1 second, task1's timer completes:
	![[Pasted image 20260401233433.png]]
7. Task1 wakes and runs to completion:
	![[Pasted image 20260401233505.png]]
	![[Pasted image 20260401233537.png]]
8. **But main doesn't wake up yet** because **main is awaiting task2, not task1**. The event loop simply saves task1's result in memory:
	![[Pasted image 20260401233634.png]]
9. After 2 seconds total, task2's timer completes:
	![[Pasted image 20260401233712.png]]
10. Task2 wakes and runs to completion:
	![[Pasted image 20260401233735.png]]
11. **Now main wakes up** because task2 (what it was awaiting) is complete:
	![[Pasted image 20260401233754.png]]
12. Main prints "Task 2 fully complete"
13. Main reaches `await task1`. Task1 is **already complete**, so the await immediately retrieves the saved result from memory:
	![[Pasted image 20260401233839.png]]
14. Main prints "Task 1 fully complete"

The key insight: the **event loop runs tasks based on readiness and scheduling order, not based on what you're currently awaiting**. Awaiting controls when your code can proceed, not which task the event loop chooses to run next.

You could even replace the await with something unrelated:
```python
async def main():
    task1 = asyncio.create_task(async_function(1))
    task2 = asyncio.create_task(async_function(2))
    
    await asyncio.sleep(2.5)  # Not awaiting either task
    print("Sleep complete")
    result1 = await task1
    result2 = await task2
    return [result1, result2]
```

Both tasks still run concurrently while main sleeps for 2.5 seconds. When the sleep completes, both tasks have already finished, and awaiting them just retrieves their saved results.
## Example 5: Blocking the Event Loop with Synchronous Code
A critical mistake is using **synchronous blocking calls inside async functions**:
```python
import asyncio
import time

async def async_function(delay):
    print(f"Doing something with {delay}")
    time.sleep(delay)  # BLOCKING CALL
    print(f"Done with {delay}")
    return f"async result of {delay}"

async def main():
    task1 = asyncio.create_task(async_function(1))
    task2 = asyncio.create_task(async_function(2))
    
    result1 = await task1
    print("Task 1 fully complete")
    result2 = await task2
    print("Task 2 fully complete")
    
    return [result1, result2]
```

The output shows:
```
Doing something with 1
Done with 1
Task 1 fully complete
Doing something with 2
Done with 2
Task 2 fully complete
['Result of 1', 'Result of 2']
Finisdhed in 3.01 seconds
```

You cannot await `time.sleep()` because it's not an awaitable object. But you can call it inside an async function. Here's what happens:
1. Both tasks are created and scheduled
	![[Pasted image 20260403105628.png]]
2. `await task1` suspends main
3. The event loop finds task1 ready
4. Task1 executes and prints "Doing something with 1"
5. Task1 reaches `time.sleep(1)`
6. `time.sleep()` kicks off background IO but **does not suspend the task**. There's no await, so the task just **blocks** at this line. **The event loop is stuck**—it **cannot move to other tasks**:
	![[Pasted image 20260403105756.png]]
7. After 1 second, the sleep completes
	![[Pasted image 20260403105819.png]]
8. Task1 continues, prints "Done with 1", and returns the result:
	![[Pasted image 20260403105856.png]]
9. The main coroutine resumes, but as mentioned earlier, the fact that it is now ready after `task1` completes does not guarantee it will run next. The event loop operates using a FIFO queue, so it prioritizes tasks based on how long they have been ready. Since `task2` became ready before the main coroutine, it has been waiting longer in the queue. As a result, the event loop schedules `task2` to run next (note: the animation in the video is missing this aspect):
	![[Pasted image 20260403110643.png]]
10. Task2 prints "Doing something with 2"
11. Task2 reaches `time.sleep(2)` and **blocks** for 2 seconds
12. After 2 seconds, task2 continues and completes:
	![[Pasted image 20260403110726.png]]
13. Main finally resumes and completes

Total time: 3 seconds. No concurrency. Both tasks start sequentially and complete sequentially. The blocking `time.sleep()` prevents the event loop from switching between tasks.

**This is exactly what happens when you run any blocking synchronous code in async functions.** This could be `requests.get()` for web requests, synchronous database queries, or any other synchronous library. The synchronous code doesn't know how to yield control back to the event loop, so it blocks everything.
## The Asyncio Philosophy: Scheduling vs. Control (*by Gemini*)
### 1. The Event Loop is an Autonomous Manager
When a task like `task1` finishes its I/O and becomes "ready," the event loop uses a **FIFO (First-In, First-Out)** strategy to decide what to run next. While it might seem important to track whether the loop returns to the `main()` coroutine or starts `task2`, in practice, you should **not** get bogged down in these details.
- **Scalability:** In real-world applications, `asyncio` handles tens or hundreds of concurrent tasks. Attempting to manually track the execution order at that scale is impossible.
- **Ready-State Execution:** The event loop is designed to simply "do its job" by executing whatever task is ready. If a task is ready to go, the loop will give it a turn when the current task yields control (usually via `await`).
### 2. The Myth of Predictability
In "toy" examples or tutorials, we often know exactly when a task will finish (e.g., `asyncio.sleep(1)`). However, **real-world asynchronous code is not "cut and dry."**
- **Unpredictable I/O:** Network latency, database response times, and file system speeds are variable.
- **Relinquishing Control:** We don't have—and shouldn't _want_—fine-grained control over exactly when a task gets its turn on the CPU. We let the loop handle the "when" so we can focus on the "what."
### 3. What We Actually Control: Dependencies
Our control as developers is limited to **logic flow**, not **scheduling**. We cannot force the event loop to pick a specific task next, but we can prevent our code from moving forward until a specific result is ready.

Key Distinction:
- **Scheduling (Loop's Job):** Deciding which "ready" task gets the next slice of CPU time.
 - **Dependency (Developer's Job):** Using `await` to ensure a coroutine does not proceed until a specific task is complete.

**Example:** If you need `task1` to finish before the next line of code runs, you must `await task1`. This doesn't tell the loop "Run task1 right now"; it tells the loop "Pause this coroutine until task1 is done, and feel free to run anything else that is ready in the meantime."

# Running Blocking Code with Threads and Processes (*To read again*)
When you must use synchronous blocking code without an async alternative, delegate it to threads or processes.
## Using Threads for Blocking IO
For blocking IO operations, convert the blocking function back to a regular synchronous function:
```python
import asyncio

def sync_function(delay):  # Regular synchronous function
    print(f"Doing something with {delay}", flush=True)
    time.sleep(delay)
    print(f"Done with {delay}", flush=True)
    return f"result of {delay}"

async def main():
    task1 = asyncio.create_task(asyncio.to_thread(sync_function, 1))
    task2 = asyncio.create_task(asyncio.to_thread(sync_function, 2))
    
    result1 = await task1
    print("Task 1 fully complete")
    result2 = await task2
    print("Task 2 fully complete")
    
    return [result1, result2]
```

The `flush=True` in print statements ensures output appears in the expected order when running across multiple threads.

Key points about `asyncio.to_thread()`:
- Pass the function itself (not called with parentheses)
- Pass arguments separately
- It wraps the synchronous function with a future and makes it awaitable
- The function executes in a separate thread
- The event loop remains free to manage other tasks

Execution flow:
1. Both tasks are created with `asyncio.to_thread()`
2. Each task is scheduled on the event loop
3. `await task1` suspends main
4. The event loop finds task1
5. Task1 hits an internal await (created by `to_thread`)
6. This kicks off a background thread running `sync_function(1)`
7. Task1 suspends, waiting for the thread to complete
8. The event loop finds task2
9. Task2 also kicks off a background thread running `sync_function(2)`
10. Task2 suspends
11. Both threads run concurrently in the background
12. When the first thread completes, task1 wakes and returns
13. Main wakes, prints completion, and awaits task2
14. When the second thread completes, task2 wakes and returns
15. Main completes

Total time: approximately 2 seconds (plus thread overhead). Both operations run concurrently despite using blocking synchronous code.
## Using Processes for CPU-Bound Work
For CPU-bound work requiring heavy computation, use processes:
```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import os

def process_single_item(data):  # CPU-intensive synchronous function
    # Heavy computation here
    return result

async def main():
    loop = asyncio.get_running_loop()
    
    with ProcessPoolExecutor(max_workers=os.cpu_count()) as executor:
        task1 = asyncio.create_task(
            loop.run_in_executor(executor, process_single_item, data1)
        )
        task2 = asyncio.create_task(
            loop.run_in_executor(executor, process_single_item, data2)
        )
        
        result1 = await task1
        result2 = await task2
        
        return [result1, result2]

if __name__ == '__main__':
    results = asyncio.run(main())
```

The `if __name__ == '__main__'` guard is required for multiprocessing. When Python spawns multiple processes, it needs to rerun the script in each new process. This check prevents infinite loops.

Key points:
- Import `ProcessPoolExecutor` from `concurrent.futures`
- Get the running loop with `asyncio.get_running_loop()`
- Use `loop.run_in_executor()` with the executor
- Pass the function and arguments separately
- Limit workers with `max_workers=os.cpu_count()`

This wraps each process in a future that you can await. Processes have more overhead than threads for spinning up and tearing down, so execution times may be slightly longer than the raw work duration.

# Scheduling and Awaiting Multiple Tasks
## Manual Task Creation
The examples so far create tasks manually and await them individually:
```python
async def main():
	task1 = asyncio.create_task(fetch_data(1))
	task2 = asyncio.create_task(fetch_data(2))
	result1 = await task1
	result2 = await task2
	print(f"task 1 and 2 awaited results {[result1, result2]}")
```

This works but doesn't scale well for many tasks. Many times we might want to create a bunch of tasks and run them all at once. We can do this either:
- `asyncio.gather()`
- `asyncio.TaskGroup()`
## Using `asyncio.gather()`
`asyncio.gather()` schedules and awaits multiple awaitables at once:
```python
async def main():
	results = await asyncio.gather(
	    fetch_data(1),
	    fetch_data(2),
	    fetch_data(3)
	)
```

You can pass coroutines directly or tasks. When **passing coroutines**:
```python
# Gather Coroutines
async def main():
	coroutines = [fetch_data(i) for i in range(1, 3)]
	results = await asyncio.gather(*coroutines, return_exceptions=True)
	print(f"Coroutine Results: {results}")
```

The asterisk unpacks the list because `gather()` expects individual arguments, not a list.

When **passing tasks**:
```python
# Gather Tasks
async def main():
	tasks = [asyncio.create_task(fetch_data(i)) for i in range(1, 3)]
	results = await asyncio.gather(*tasks, return_exceptions=True)
	print(f"Task Results: {results}")
```

When should you use coroutines versus tasks with `gather`? If you only need results, passing coroutines directly is fine. But tasks add extra functionality—if you want to monitor or interact with tasks before they complete (checking status, cancelling them, etc.), create tasks explicitly.
### Critical: `gather()` and Error Handling
The `return_exceptions` parameter is critical. The default is `return_exceptions=False`, which is problematic:
```python
# PROBLEMATIC - default behavior
results = await asyncio.gather(*tasks)  # return_exceptions defaults to False
```

With `return_exceptions=False`:
- If one task fails, `gather()` raises the first exception immediately
- You don't get results from successful tasks
- You don't get information about other exceptions
- Other tasks are NOT cancelled—they keep running
- You risk orphaned tasks that complete without being awaited

**Always use `return_exceptions=True`:**
```python
results = await asyncio.gather(*tasks, return_exceptions=True)
```

With `return_exceptions=True`:
- Every awaitable in the gather finishes, whether it succeeds or fails
- The result is a list where each position contains either the result (on success) or the exception (on failure)
- All tasks run to completion

Use this when you want all tasks to complete regardless of individual failures. For example, crawling multiple URLs where one failure shouldn't stop the others:
```python
urls = [url1, url2, url3, ...]
results = await asyncio.gather(
    *[fetch_url(url) for url in urls],
    return_exceptions=True
)

for i, result in enumerate(results):
    if isinstance(result, Exception):
        print(f"URL {i} failed: {result}")
    else:
        print(f"URL {i} succeeded: {result}")
```
## Using `asyncio.TaskGroup()`
Task groups provide better error handling and cleanup:
```python
async def main():
	async with asyncio.TaskGroup() as tg:
	    tasks = [
	        tg.create_task(fetch_data(i))
	        for i in range(1, 3)
	    ]
	    # All tasks are awaited when the context manager exist.
	print(f"Task Group Results: {results = [task.result() for task in tasks]}")
```

Task groups are **async context managers**. Just as functions can be async when they need to perform IO operations, context managers can be async when they need to perform async work during setup or teardown.

Key points about task groups: 
- You create tasks with `tg.create_task()` inside the context manager
- You don't await anything explicitly
- The context manager automatically awaits all created tasks when exiting
- You retrieve results after exiting the context manager with `task.result()`

The task group handles tracking tasks, waiting for completions, handling cancellations, and managing errors during both entry and exit.
## `gather()` vs Task Groups: Error Handling
The critical difference is error handling behavior.

**With `asyncio.gather(return_exceptions=False)` (default, not recommended):**
- On first failure, raises the exception immediately
- Doesn't cancel other tasks
- Risks orphaned tasks
- You get no information about successful tasks or other failures

**With `asyncio.gather(return_exceptions=True)`:**
- All tasks run to completion regardless of failures
- Results list contains either results or exceptions
- Use when you want every task to finish even if some fail

**With Task Groups:**
- On first failure, immediately cancels all other tasks
- Raises an `ExceptionGroup` containing all exceptions (from failed tasks and cancelled tasks)
- No option to keep running after a failure
- Use when you want all-or-nothing behavior—tasks either all succeed together or fail together

Choosing between them:
```python
# Want tasks to continue even if some fail (e.g., crawling URLs)
results = await asyncio.gather(*tasks, return_exceptions=True)

# Want all tasks to succeed together or fail together (e.g., multi-step transaction)
async with asyncio.TaskGroup() as tg:
    tasks = [tg.create_task(operation(i)) for i in range(10)]
```

There's rarely a good reason to use `gather()` with `return_exceptions=False`. If you want fail-fast behavior with proper cleanup, use task groups. If you want tasks to continue regardless of failures, use `gather(return_exceptions=True)`.

# Real-World Example: Optimizing an Image Downloader
Consider a script that downloads and processes images. The initial synchronous version:
```python
import time
from pathlib import Path

import requests
from PIL import Image

IMAGE_URLS = [
    "https://images.unsplash.com/photo-1516117172878-fd2c41f4a759?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1532009324734-20a7a5813719?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1524429656589-6633a470097c?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1530224264768-7ff8c1789d79?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1564135624576-c5c88640f235?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1541698444083-023c97d3f4b6?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1522364723953-452d3431c267?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1493976040374-85c8e12f0c0e?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1530122037265-a5f1f91d3b99?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1516972810927-80185027ca84?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1550439062-609e1531270e?w=1920&h=1080&fit=crop",
    "https://images.unsplash.com/photo-1549692520-acc6669e2f0c?w=1920&h=1080&fit=crop",
]


ORIGINAL_DIR = Path("original_images")
PROCESSED_DIR = Path("processed_images")


def download_single_image(session: requests.Session, url: str, img_num: int) -> Path:
    print(f"Downloading {url}...")
    ts = int(time.time())
    url = f"{url}?ts={ts}"  # Add timestamp to avoid caching issues
    response = session.get(url, timeout=10, allow_redirects=True)
    response.raise_for_status()

    filename = f"image_{img_num}.jpg"
    download_path = ORIGINAL_DIR / filename

    with download_path.open("wb") as f:
        for chunk in response.iter_content(chunk_size=8192):
            f.write(chunk)

    print(f"Downloaded and saved to: {download_path}")
    return download_path


def download_images(urls: list) -> list[Path]:
    with requests.Session() as session:
        img_paths = [
            download_single_image(session, url, img_num)
            for img_num, url in enumerate(urls, start=1)
        ]

    return img_paths


def process_single_image(orig_path: Path) -> Path:
    save_path = PROCESSED_DIR / orig_path.name

    with Image.open(orig_path) as img:
        data = list(img.getdata())
        width, height = img.size
        new_data = []

        for i in range(len(data)):
            current_r, current_g, current_b = data[i]

            total_diff = 0
            neighbor_count = 0

            for dx, dy in [(1, 0), (0, 1)]:
                x = (i % width) + dx
                y = (i // width) + dy

                if 0 <= x < width and 0 <= y < height:
                    neighbor_r, neighbor_g, neighbor_b = data[y * width + x]
                    diff = (
                        abs(current_r - neighbor_r)
                        + abs(current_g - neighbor_g)
                        + abs(current_b - neighbor_b)
                    )
                    total_diff += diff
                    neighbor_count += 1

            if neighbor_count > 0:
                edge_strength = total_diff // neighbor_count
                if edge_strength > 30:
                    new_data.append((255, 255, 255))
                else:
                    new_data.append((0, 0, 0))
            else:
                new_data.append((0, 0, 0))

        edge_img = Image.new("RGB", (width, height))
        edge_img.putdata(new_data)
        edge_img.save(save_path)

    print(f"Processed {orig_path} and saved to {save_path}")
    return save_path


def process_images(orig_paths: list[Path]) -> list[Path]:
    img_paths = [process_single_image(orig_path) for orig_path in orig_paths]

    return img_paths


def main():
    ORIGINAL_DIR.mkdir(parents=True, exist_ok=True)
    PROCESSED_DIR.mkdir(parents=True, exist_ok=True)

    start_time = time.perf_counter()

    img_paths = download_images(IMAGE_URLS)

    proc_start_time = time.perf_counter()

    processed_paths = process_images(img_paths)

    finished_time = time.perf_counter()

    dl_total_time = proc_start_time - start_time
    proc_total_time = finished_time - proc_start_time
    total_time = finished_time - start_time

    print(
        f"\nDownloaded {len(img_paths)} images in: {dl_total_time:.2f} seconds. {(dl_total_time / total_time) * 100:.2f}% of total time",
    )
    print(
        f"Processed {len(processed_paths)} images in: {proc_total_time:.2f} seconds. {(proc_total_time / total_time) * 100:.2f}% of total time",
    )
    print(
        f"\nTotal execution time: {total_time:.2f} seconds. {(total_time / total_time) * 100:.2f}% of total time",
    )


if __name__ == "__main__":
    main()
```

Baseline execution: 23 seconds total (13 seconds downloading, 10 seconds processing).
## Step 1: Profile to Identify Bottlenecks
Before optimizing, profile the code to identify what's IO-bound versus CPU-bound:
```bash
python -m scalene --html --outfile profile_report.html script.py
```

Scalene breaks execution time into:
- **Python time**: Time spent executing Python code (typically CPU-bound)
- **Native time**: Time spent in native C code
- **System time**: Time spent waiting on the system (typically IO-bound)

The profile report shows:
- Most system time is in `download_single_image()`—this is IO-bound
- Most Python time is in `process_single_image()`—this is CPU-bound

This tells you:
- Downloads benefit from async/IO or threading
- Processing benefits from multiprocessing
## Step 2: First Optimization with Threads
If continuing with the synchronous `requests` library (which isn't async-compatible), use threads:
```python
import asyncio

def download_single_image(url: str, img_num: int) -> Path:
    print(f"Downloading {url}...")
    ts = int(time.time())
    url = f"{url}?ts={ts}"  # Add timestamp to avoid caching issues
    response = request.get(url, timeout=10, allow_redirects=True)
    response.raise_for_status()

    filename = f"image_{img_num}.jpg"
    download_path = ORIGINAL_DIR / filename

    with download_path.open("wb") as f:
        for chunk in response.iter_content(chunk_size=8192):
            f.write(chunk)

    print(f"Downloaded and saved to: {download_path}")
    return download_path
    
    
async def download_images(urls: list) -> list[Path]:
	async with asyncio.TaskGroup() as tg:
	tasks = [
		tg.create_task(asyncio.to_thread(download_single_image, url, img_num))
		for img_num, urk in ernumerate(urls, start=1)
	]

	img_paths = [task.result() for task in tasks]

    return img_paths
```

Changes made:
- Removed the session (might not be thread-safe)
- Kept `download_single_image()` as synchronous
- Made `download_images()` async
- Created tasks using `asyncio.to_thread()` for each URL
- Used a task group to manage all tasks

For processing, threading won't help (CPU-bound work), but test it anyway to demonstrate:
```python
async def process_images(orig_paths: list[Path]):
    async with asyncio.TaskGroup() as tg:
        tasks = [
            tg.create_task(asyncio.to_thread(process_single_image, orig_path))
            for orig_path in orig_paths
        ]
    img_paths = [task.result() for task in tasks]
    
    return img_paths
```

Update main to be async:
```python
async def main():
    image_paths = await download_images(IMAGE_URLS)
    processed_paths = await process_images(image_paths)
    return processed_paths

if __name__ == '__main__':
    results = asyncio.run(main())
```

Results:
- Total time: approximately 14 seconds
- Downloads: 2 seconds (down from 13 seconds)
- Processing: 10.63 seconds (no improvement from 10 seconds)

Threading **dramatically improved IO-bound downloads** but didn't help CPU-bound processing, as expected.
## Step 3: Full Optimization with Async Libraries and Multiprocessing
Right now, we’re pushing all work onto threads. However, that approach should be reserved for cases where no asynchronous alternative exists. For request handling, we already have **asyncio-compatible libraries**, with the most common options being:
- For HTTP: `httpx` or `aiohttp`
- For file IO: `aiofiles`

Install them:
```bash
uv add httpx aiofiles
```

Rewrite downloads with async libraries:
```python
import asyncio
import httpx
import aiofiles

async def download_single_image(client: httpx.AsyncClient, url: str, img_num: int) -> Path:
	print(f"Downloading {url}...")
	ts = int(time.time())
	url = f"{url}?ts={ts}"
	response = await client.get(url, timeout=10, follow_redirects=True)
	response.raise_for_status()
	
	filename = f"image_{img_num}.jpg"
	download_path = ORIGINAL_DIR / filename
	
	async with aiofiles.open(download_path, "wb") as f:
		async for chunk in response.aiter_bytes(chunk_size=8192):
			await f.write(chunk)
	
	print(f"Downloaded and saved to {download_path}")
	return download_path


async def download_images(urls):
    async with httpx.AsyncClient() as client:
        async with asyncio.TaskGroup() as tg:
            tasks = [
                tg.create_task(download_single_image(client, url, img_num))
                for img_num, url in enumerate(urls, start=1)
            ]
    return [task.result() for task in tasks]
```

Key changes:
- `download_single_image()` is now async
- Use `async with` for the HTTP response (async context manager)
- Use `async with` for file operations with `aiofiles`
- Use `async for` to iterate over response chunks

The `async for` loop is an asynchronous iterator. With a regular iterator, the `for` keyword pulls the next value immediately. With an async iterator, each next value may involve waiting for IO (network data arriving). The `async for` syntax handles this by implicitly awaiting each iteration.

`response.aiter_bytes()` provides an async iterator. Each loop iteration performs an implicit await to get the next chunk. Inside the loop, `await f.write(chunk)` is necessary because file writing is also asynchronous with `aiofiles`.

For processing, use multiprocessing:
```python
from concurrent.futures import ProcessPoolExecutor
import os

async def process_images(orig_paths: list[Path]):
    loop = asyncio.get_running_loop()
    
    with ProcessPoolExecutor() as executor:
        tasks = [
            loop.run_in_executor(executor, process_single_image, orig_path)
            for orig_path in orig_paths
        ]
        return await asyncio.gather(*tasks, return_exceptions=True)
```

Changes:
- Import `ProcessPoolExecutor` and `os`
- Get CPU count with `os.cpu_count()`
- Get the running event loop
- Use `ProcessPoolExecutor` with `max_workers` set to CPU count
- Create tasks with `loop.run_in_executor()`
- Use `gather()` with `return_exceptions=True`

Results:
- Total time: approximately 5 seconds
- Downloads: 1.6 seconds (down from 13 seconds)
- Processing: 3.25 seconds (down from 10 seconds)

![[Pasted image 20260405213043.png]]

This is approximately a 4.7x speedup overall.
## Step 4: Adding Rate Limits and Resource Management
Running thousands of concurrent operations can overwhelm your machine or hammer servers. Add **rate limiting** with **semaphores**:
```python
DOWNLOAD_LIMIT = 4

async def download_images(urls):
    download_semaphore = asyncio.Semaphore(DOWNLOAD_LIMIT)
    
    async def download_with_limit(url, client):
        async with download_semaphore:
            return await download_single_image(url, client)
    
    async with httpx.AsyncClient() as client:
        async with asyncio.TaskGroup() as tg:
            tasks = [
                tg.create_task(download_with_limit(url, client))
                for url in urls
            ]
    return [task.result() for task in tasks]
```

The semaphore limits concurrent downloads to four. When you enter `async with download_semaphore`, it acquires a slot. If four slots are already acquired, it waits until one releases. When you exit the context manager, it releases the slot, allowing another download to proceed.

Execution flow with semaphore:
1. All tasks are created and scheduled
2. First four downloads acquire semaphore slots and start
3. When one download completes and exits the context manager, it releases its slot
4. The next waiting download acquires the slot and proceeds
5. This continues until all downloads complete

For processes, the `ProcessPoolExecutor` already has a built-in limit with `max_workers`:
```python
with ProcessPoolExecutor(max_workers=os.cpu_count()) as executor:
```

This limits concurrent processes to your CPU count, preventing resource exhaustion.

Alternative: You could also add a semaphore for processes:
```python
async def process_images(image_paths):
    loop = asyncio.get_running_loop()
    process_semaphore = asyncio.Semaphore(CPU_WORKERS)
    
    async def process_with_limit(path, executor):
        async with process_semaphore:
            return await loop.run_in_executor(executor, process_single_image, path)
    
    with ProcessPoolExecutor() as executor:
        tasks = [
            process_with_limit(path, executor)
            for path in image_paths
        ]
        return await asyncio.gather(*tasks, return_exceptions=True)
```

With these limits, the script is slightly slower (due to not running everything simultaneously) but much more responsible with resource usage. Even with thousands of images, you maintain controlled concurrency.

# Common Pitfalls and How to Avoid Them
## Pitfall 1: Forgetting to Await
```python
async def main():
    task1 = asyncio.create_task(fetch_data(1))
    task2 = asyncio.create_task(fetch_data(2))
    # Missing: await task1, await task2
    return None
```

Tasks are  **created and scheduled but never awaited**. Depending on timing, they might not run at all, or they might run but get cancelled when the  event loop closes. You won't get errors—tasks just silently fail to complete.

Always await your tasks:
```python
async def main():
    task1 = asyncio.create_task(fetch_data(1))
    task2 = asyncio.create_task(fetch_data(2))
    result1 = await task1
    result2 = await task2
    return [result1, result2]
```

Or use gather/task groups that handle awaiting for you.
## Pitfall 2: Script Ending Before Tasks Complete
```python
async def main():
    task1 = asyncio.create_task(fetch_data(1))
    task2 = asyncio.create_task(fetch_data(2))
    
    result1 = await task1
    print('Task 1 fully completed')
    result2 = await asyncio.sleep(0.1)  # Only sleeping 0.1 seconds
    # task2 takes 2 seconds - it won't finish!
    print("Task 2 fully completed")
    
    return [result1, result2]  # Missing task2 result
```

![[Pasted image 20260405214815.png]]

If you await something other than your tasks, and that something completes before your tasks finish, your tasks get abandoned. Always **ensure all tasks are awaited before the script ends**.
## Pitfall 3: Blocking the Event Loop
```python
async def fetch_data(delay):
    time.sleep(delay)  # WRONG - blocks event loop
    return result

# Should be:
async def fetch_data(delay):
    await asyncio.sleep(delay)  # Correct - yields control
    return result
```

Any synchronous blocking call (`time.sleep()`, `requests.get()`, synchronous database queries, synchronous file IO) blocks the entire event loop. Use async alternatives or delegate to threads/processes.
## Pitfall 4: Using Gather Without return_exceptions=True
```python
# PROBLEMATIC
results = await asyncio.gather(*tasks)  # Can leave orphaned tasks

# BETTER
results = await asyncio.gather(*tasks, return_exceptions=True)

# OR use task groups for fail-together behavior
async with asyncio.TaskGroup() as tg:
    tasks = [tg.create_task(op()) for op in operations]
```
## Tools to Catch These Mistakes
**Use a good linter.** Ruff with proper async/await configuration catches many async mistakes:
```toml
# pyproject.toml
[tool.ruff]
select = ["ASYNC"]  # Enable async-specific checks
```

**Enable debug mode during development:**
```python
asyncio.run(main(), debug=True)
```

This enables additional warnings and checks:
- Warns about unawaited coroutines
- Provides more detailed error messages
- Helps identify blocking operations
- Shows slow callbacks

Only use `debug=True` in development—it adds overhead.

# When to Use Async, Threads, or Processes
## Use Async/IO For:
- **IO-bound work with async libraries available**
- Network requests (with `httpx`, `aiohttp`)
- Database queries (with async drivers like `asyncpg`, `aiomysql`)
- File operations (with `aiofiles`)
- Web frameworks (`FastAPI`, `Starlette`)
- Any waiting on external operations with async support
## Use Threads For:
- **IO-bound work when async libraries aren't available**
- Synchronous database drivers without async alternatives
- Third-party libraries that only provide synchronous APIs
- `requests` library (until you can migrate to `httpx`)
- Any synchronous IO operation that would otherwise block
## Use Multiprocessing For:
- **CPU-bound work requiring heavy computation**
- Image processing
- Video encoding
- Cryptographic operations
- Machine learning inference
- Data transformation and analysis
- Scientific computing
- Any operation that maxes out CPU rather than waiting on IO
## How to Determine Which Type of Work You Have
**Method 1: Intuition based on operation type**
- Words like "fetch", "get", "request", "query", "read", "write" usually indicate IO-bound
- Words like "compute", "calculate", "process", "encode", "transform" usually indicate CPU-bound

**Method 2: Profiling with tools like Scalene**
- High **system time** indicates IO-bound (waiting on external operations)
- High **Python time** indicates CPU-bound (executing Python code)
- High **native time** indicates CPU-bound (executing C extensions)

**Method 3: Test both approaches**
- If threading speeds it up significantly: IO-bound
- If threading doesn't help but multiprocessing does: CPU-bound
- If neither helps: check for blocking synchronous code in async functions

# The Growing Async Ecosystem
Async/IO libraries continue to expand:

**Web Frameworks:**
- FastAPI (async-first web framework)
- Starlette (async web toolkit)
- Quart (async Flask alternative)
- Sanic (async web server)

**HTTP Clients:**
- httpx (async requests alternative)
- aiohttp (mature async HTTP client/server)

**Database Drivers:**
- asyncpg (PostgreSQL)
- aiomysql (MySQL)
- motor (MongoDB)
- aiosqlite (SQLite)
- SQLAlchemy 1.4+ (async support)

**File Operations:**
- aiofiles (async file IO)

**Task Queues:**
- arq (async task queue)
- dramatiq (async task processing)

**Testing:**
- pytest-asyncio (async test support)

The ecosystem continues growing, making async/IO increasingly practical for production applications. As more libraries adopt async/IO, you can write more of your application using async patterns, improving concurrency and resource utilization without the complexity of managing threads and processes manually.