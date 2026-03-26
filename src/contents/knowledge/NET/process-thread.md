---
id: process-thread
topic: Process and Thread
category: .NET
---
## Simple idea

* A **process** is a running application.
* A **thread** is a path of execution inside that application.

Example:

* When you open **Chrome**, Chrome is a **process**
* Inside Chrome, many **threads** may exist:

  * one for UI
  * one for network work
  * one for rendering
  * one for background tasks

So:

* **Process = container**
* **Thread = worker inside that container**

#  What is a process?

A **process** is an instance of a program that is currently running.

A process has its own:

* memory space
* resources
* loaded libraries
* file handles
* security context

In C#, your `.exe` or running app becomes a process when started.

Example:

* Your ASP.NET app running on the server = one process
* Your console app when launched = one process

You can access process info in C# with `System.Diagnostics.Process`.

Example:

```csharp
using System.Diagnostics;

var current = Process.GetCurrentProcess();

Console.WriteLine($"Process Name: {current.ProcessName}");
Console.WriteLine($"Process Id: {current.Id}");
```

---

#  What is a thread?

A **thread** is the smallest unit of execution inside a process.

A process can have:

* one thread
* or many threads

Each thread runs code independently.

For example, a thread can:

* calculate data
* call an API
* write logs
* handle user input

All threads inside the same process usually **share the same process memory**, which makes communication easy, but also creates risks like:

* race conditions
* deadlocks
* shared state bugs

---

#  Main difference

## Process

* Heavyweight
* Has its own memory and resources
* Is isolated from other processes
* More expensive to create

## Thread

* Lightweight compared to a process
* Lives inside a process
* Shares process memory with other threads in that process
* Cheaper to create than a process

---


#  In C#, where do threads show up?

In C#, threads appear in several ways:

* `Thread`
* `ThreadPool`
* `Task`
* `async/await`

Important point:

In modern C#, you usually use **Task** and **async/await** more often than creating raw threads manually.


#  Why not create threads everywhere?

Creating raw threads manually is usually not the best approach because:

* they cost memory
* they cost scheduling overhead
* too many threads can hurt performance
* managing them is harder

That is why .NET gives better abstractions like:

* `Task`
* thread pool
* `async/await`

---

# Thread pool

.NET has a **thread pool**, which is a pool of reusable worker threads.

Instead of creating a brand new thread every time, .NET can reuse an existing one.

This is faster and more efficient.

Example with `Task`:

```csharp
Task.Run(() =>
{
    Console.WriteLine("Running on a thread pool thread");
});
```

This usually uses a thread from the thread pool.

---

# `async/await` is not the same as “new thread”

A very important point:

**`async/await` does not always create a new thread.**

Example:

```csharp
await Task.Delay(1000);
```

This does **not** mean:

* one thread sleeps for 1 second

Instead, the method pauses, and the thread can return to do other work. Later, execution resumes.

So:

* **Threading** is about execution on threads
* **async/await** is about non-blocking asynchronous flow

They are related, but not identical.

---

# Shared memory and thread safety

Threads in the same process share memory.

That means this can be dangerous:

```csharp
int counter = 0;

Parallel.For(0, 1000, i =>
{
    counter++;
});

Console.WriteLine(counter);
```

You may expect `1000`, but you might get a smaller number because multiple threads update the same variable at the same time.

Safer version:

```csharp
int counter = 0;

Parallel.For(0, 1000, i =>
{
    Interlocked.Increment(ref counter);
});

Console.WriteLine(counter);
```

Tools used for thread safety include:

* `lock`
* `Monitor`
* `Mutex`
* `SemaphoreSlim`
* `Interlocked`
* concurrent collections

---

# Process vs thread in practical backend work

In a backend C# app:

* Your API app is usually one **process**
* That process handles many requests
* Requests may run on different **threads** from the thread pool

For example in ASP.NET Core:

* multiple HTTP requests can be handled concurrently
* the runtime schedules work across available threads

You normally do not manually create threads for each request.

---


#  Can processes share memory?

Not directly like threads do.

Processes are isolated from each other.

To communicate between processes, you usually need things like:

* files
* sockets
* pipes
* message queues
* shared databases
* named pipes
* gRPC / HTTP calls

This isolation makes processes safer than threads, but communication is more expensive.

---

#  When would you use each?

## Use process when:

* you want isolation
* you are running separate applications
* one crash should not directly corrupt another app’s memory

## Use thread when:

* you want concurrent work inside the same application
* you need faster communication through shared memory
* tasks belong to the same app

In normal C# app development, you usually do not say:

* “I will create processes”

You more often work with:

* one process
* many tasks/threads inside it

---

# Summary

> A process is a running application with its own memory and resources. A thread is a unit of execution inside that process. A process can have multiple threads, and those threads share the process memory. Processes are more isolated and heavier, while threads are lighter and used for concurrent work inside the same application. In modern C#, we usually prefer `Task` and `async/await` over manually creating threads.

---

#  Multithreading vs async/await

#  How threads work in `async/await`

The most important thing to know is:

**`async/await` is not the same as multithreading.**

A lot of people think:

* `async` = run on another thread

That is **not always true**.

## Simple idea

`async/await` is mainly about:

* not blocking a thread while waiting
* making better use of threads
* handling long-running I/O work efficiently

Examples of I/O work:

* calling a database
* calling an API
* reading a file
* waiting for network response

---

## Traditional blocking example

```csharp
var result = httpClient.GetStringAsync(url).Result;
Console.WriteLine(result);
```

This blocks the current thread until the result comes back.

If this is a UI app:

* the UI can freeze

If this is a server app:

* that thread cannot serve other requests while waiting

---

## Async version

```csharp
var result = await httpClient.GetStringAsync(url);
Console.WriteLine(result);
```

Now the thread does not have to sit there doing nothing.

What usually happens:

1. method starts running on a thread
2. it reaches `await`
3. if the async operation is not finished yet, the method pauses
4. the thread is released to do other work
5. when the operation completes, the method continues later

So `await` often means:

* “pause here”
* “continue when the work is done”
* “do not block the current thread while waiting”

---

## Does `await` create a new thread?

Usually, **no**.

Example:

```csharp
await Task.Delay(2000);
```

This does **not** mean:

* a thread sleeps for 2 seconds

Instead:

* a timer is set
* the method returns control
* no thread is wasted during the wait
* later, continuation resumes

This is why async is efficient.

---

## What thread resumes after `await`?

That depends on the application type.

### In UI apps

Such as:

* WinForms
* WPF

After `await`, code often resumes on the **original UI thread**.

That is useful because UI controls usually must be updated from the UI thread.

Example:

```csharp
private async void Button_Click(object sender, EventArgs e)
{
    label1.Text = "Loading...";

    var text = await httpClient.GetStringAsync("https://example.com");

    label1.Text = text;
}
```

This works because after `await`, execution usually continues on the UI thread.

---

### In ASP.NET Core

ASP.NET Core does **not** have a UI thread or classic synchronization context like desktop UI apps.

So after `await`:

* continuation may run on any thread pool thread

Usually you do not care which exact thread it resumes on.

---

## Example flow

```csharp
public async Task DoWorkAsync()
{
    Console.WriteLine("Start");
    await Task.Delay(1000);
    Console.WriteLine("End");
}
```

Conceptually:

* `Start` runs on some thread
* `await Task.Delay(1000)` pauses method
* thread is free to do something else
* after 1 second, method continues
* continuation may run on:

  * same thread
  * or another thread
    depending on environment

So the key idea is:

**`async/await` manages continuation of work, not “one thread dedicated to one method”.**

---

## CPU-bound work is different

If your work is CPU-heavy, like:

* image processing
* large calculations
* encryption
* parsing huge data

Then `async/await` alone does not magically make it non-blocking.

Example:

```csharp
public async Task<int> CalculateAsync()
{
    int sum = 0;
    for (int i = 0; i < 100000000; i++)
    {
        sum += i;
    }

    return sum;
}
```

This method is marked `async`, but there is no `await`.
It still runs synchronously on the current thread.

For CPU work, you may offload to another thread:

```csharp
var result = await Task.Run(() => HeavyCalculation());
```

Here:

* `Task.Run` uses a thread pool thread
* this is closer to actual multithreading

So:

* **I/O-bound async**: often no dedicated extra thread during waiting
* **CPU-bound with `Task.Run`**: usually uses another thread

---

## Easy mental model

### `await` with I/O

“I started something that takes time, but I do not want to block this thread while waiting.”

### `Task.Run`

“I want this work to run on another thread.”

That is a very useful interview distinction.

---

#  What is multithreading?

**Multithreading** means a process uses multiple threads to do work concurrently.

Instead of one thread doing everything one by one, multiple threads can work at the same time or take turns quickly.

---

## Example

Imagine one restaurant kitchen.

### Single-threaded

One cook does:

* take order
* chop vegetables
* cook food
* wash dishes
* serve customer

Everything is sequential.

### Multithreaded

Multiple cooks do different work:

* one cooks
* one prepares ingredients
* one plates food

More work can happen concurrently.

---

## In C#

A process can have many threads.

Example:

* main thread
* thread pool worker threads
* GC thread
* finalizer thread
* custom worker threads

You can create multithreading in many ways:

* `new Thread(...)`
* `Task.Run(...)`
* `Parallel.For(...)`
* thread pool APIs

---

## Simple example with multiple threads

```csharp
using System.Threading;

Thread t1 = new Thread(() =>
{
    Console.WriteLine("Thread 1 working");
});

Thread t2 = new Thread(() =>
{
    Console.WriteLine("Thread 2 working");
});

t1.Start();
t2.Start();
```

Now two threads can run independently.

---

#  Multithreading vs async/await

This is the most important distinction.

## Multithreading

About **multiple threads** doing work.

## Async/await

About **non-blocking asynchronous flow**.

They can work together, but they are not the same thing.

---

## Compare them

### Multithreading example

You have 4 threads doing 4 calculations at once.

### Async example

You start 4 HTTP calls and await them without blocking threads while waiting for network responses.

In async I/O, there may not be 4 busy threads the whole time.

That is why async scales well for I/O-heavy apps.

---

#  Why use multithreading?

Use multithreading when you want to:

* do CPU work in parallel
* keep UI responsive while background work runs
* process multiple items faster
* use multiple CPU cores

Example:

* resize 1000 images faster
* process multiple files in parallel
* run independent calculations together

---


#  Practical examples

## Example A: API call

Best fit: `async/await`

```csharp
var response = await httpClient.GetAsync(url);
```

Reason:

* network call is I/O-bound
* do not block a thread while waiting

---

## Example B: heavy calculation

Best fit: multithreading or parallelism

```csharp
var result = await Task.Run(() => HeavyCalculation());
```

Reason:

* CPU-bound work
* move work off current thread

---

## Example C: process many independent items

Best fit: parallel work

```csharp
Parallel.ForEach(items, item =>
{
    ProcessItem(item);
});
```

Reason:

* multiple CPU cores can help

---


# Summary

## `async/await`

Use it for:

* HTTP calls
* database calls
* file I/O
* waiting for external systems

Goal:

* avoid blocking threads

## Multithreading

Use it for:

* CPU-heavy work
* parallel processing
* work that should run on multiple threads

Goal:

* do more compute work concurrently

---

> `async/await` is mainly for asynchronous, non-blocking operations, especially I/O-bound work. It does not automatically mean a new thread is created. When an awaited operation is in progress, the current thread can be released to do other work, and the method continues later.
> Multithreading means using multiple threads in the same process to execute work concurrently or in parallel. It is commonly used for CPU-bound work, but it introduces complexity like race conditions and synchronization.

---

* **Waiting for something external** -> use `async/await`
* **Doing heavy computation** -> think about multithreading / parallelism

---

# Comparison table

| Topic                     | `async/await`           | Multithreading               |
| ------------------------- | ----------------------- | ---------------------------- |
| Main purpose              | Non-blocking async flow | Run work on multiple threads |
| Best for                  | I/O-bound work          | CPU-bound work               |
| Always uses extra thread? | No                      | Yes, by definition           |
| Complexity                | Usually simpler         | Harder                       |
| Common risk               | Misunderstanding flow   | Race conditions, deadlocks   |

# Task.Run, Task.WhenAll, and Parallel.ForEach

# Quick summary

| Feature                        | `Task.Run`                                    | `Task.WhenAll`                         | `Parallel.ForEach`                            |
| ------------------------------ | --------------------------------------------- | -------------------------------------- | --------------------------------------------- |
| Main purpose                   | Run one piece of work on a thread pool thread | Wait for many tasks together           | Process a collection in parallel              |
| Best for                       | Offloading CPU work                           | Coordinating multiple async operations | CPU-bound work over many items                |
| Usually uses multiple threads? | Usually yes                                   | Not necessarily                        | Usually yes                                   |
| Good for async I/O?            | Usually no need                               | Yes                                    | No                                            |
| Good for CPU-bound loops?      | Sometimes                                     | Not by itself                          | Yes                                           |
| Returns                        | `Task` or `Task<T>`                           | Combined task for all tasks            | No task in classic version; blocks until done |
| Common interview idea          | “Run this work in background thread pool”     | “Start many tasks, await all”          | “Parallelize collection processing”           |

---

#  `Task.Run`

## What it means

`Task.Run` says:

> Put this work onto the thread pool and run it asynchronously.

Example:

```csharp
var result = await Task.Run(() => HeavyCalculation());
```

This usually means:

* current thread starts the task
* thread pool thread does the calculation
* caller can await the result

## Best use case

Use `Task.Run` for **CPU-bound work**, such as:

* heavy calculation
* image processing
* file parsing
* compression

## Example

```csharp
int HeavyCalculation()
{
    Thread.Sleep(2000);
    return 42;
}

var result = await Task.Run(() => HeavyCalculation());
Console.WriteLine(result);
```

## Key point

`Task.Run` is about **one unit of work**.

---

#  `Task.WhenAll`

## What it means

`Task.WhenAll` says:

> I already have multiple tasks. Wait until all of them finish.

Example:

```csharp
var task1 = httpClient.GetStringAsync("https://site1.com");
var task2 = httpClient.GetStringAsync("https://site2.com");
var task3 = httpClient.GetStringAsync("https://site3.com");

await Task.WhenAll(task1, task2, task3);
```

## Best use case

Use it for **multiple async operations**, especially I/O:

* multiple API calls
* multiple DB calls
* multiple file reads
* multiple service calls

## Key point

`Task.WhenAll` does **not** itself mean “use multiple threads”.

It just waits for many tasks together.

If the tasks are:

* HTTP calls -> usually mostly async I/O, not dedicated busy threads
* `Task.Run(...)` CPU work -> then multiple threads are likely involved

So `Task.WhenAll` is more about **coordination** than execution style.

---

#  `Parallel.ForEach`

## What it means

`Parallel.ForEach` says:

> Take this collection and process multiple items in parallel.

Example:

```csharp
Parallel.ForEach(files, file =>
{
    ProcessFile(file);
});
```

## Best use case

Use it for **CPU-bound work over many independent items**:

* processing many images
* transforming data
* calculating hashes
* evaluating many records

## Key point

`Parallel.ForEach` is built for **data parallelism**.

It usually uses multiple thread pool threads and tries to use CPU cores efficiently.

---

# Clear mental model

## `Task.Run`

“Run this one heavy thing on another thread pool thread.”

## `Task.WhenAll`

“I have many tasks. Let them all progress, then continue when all are done.”

## `Parallel.ForEach`

“I have many items. Split them up and process them in parallel.”

---

#  Side-by-side example

Suppose you have 100 files.

## A. `Task.Run`

Use this when you want to offload one big job:

```csharp
await Task.Run(() =>
{
    foreach (var file in files)
    {
        ProcessFile(file);
    }
});
```

Meaning:

* one background task
* usually one worker thread doing the loop
* not parallel per item

So this is **not the same** as parallelizing the whole collection.

---

## B. `Task.WhenAll`

Use this when each file has an async method:

```csharp
var tasks = files.Select(file => UploadFileAsync(file));
await Task.WhenAll(tasks);
```

Meaning:

* start many async uploads
* wait for all uploads
* best when upload is I/O-bound

---

## C. `Parallel.ForEach`

Use this when processing files is CPU-heavy:

```csharp
Parallel.ForEach(files, file =>
{
    ProcessFile(file);
});
```

Meaning:

* process multiple files at once
* usually multiple threads
* good for CPU work

---



#  Important differences

## `Task.Run` vs `Parallel.ForEach`

Both often use thread pool threads.

But:

* `Task.Run` = one task
* `Parallel.ForEach` = many items processed in parallel

Example:

```csharp
await Task.Run(() => ProcessOneBigJob());
```

vs

```csharp
Parallel.ForEach(items, item => ProcessItem(item));
```

So `Parallel.ForEach` is more specialized for collection parallelism.

---

## `Task.WhenAll` vs `Parallel.ForEach`

These are commonly confused.

### `Task.WhenAll`

* best for async tasks
* especially I/O-bound work
* non-blocking style

### `Parallel.ForEach`

* best for CPU-bound synchronous work
* parallel loop style

Bad mismatch:

```csharp
Parallel.ForEach(urls, url =>
{
    var html = httpClient.GetStringAsync(url).Result;
});
```

This blocks threads and is usually a poor design.

Better:

```csharp
var tasks = urls.Select(url => httpClient.GetStringAsync(url));
var results = await Task.WhenAll(tasks);
```

---

#  Does each create a new thread?

## `Task.Run`

Usually uses a thread pool thread, not a brand-new thread every time.

## `Task.WhenAll`

No. It just waits for tasks. It does not inherently create threads.

## `Parallel.ForEach`

Usually uses multiple thread pool threads, but not one new thread per item.

---

#  Blocking vs non-blocking

## `Task.Run`

Can be awaited, so caller can be non-blocking.
But the work inside is still usually synchronous CPU work.

## `Task.WhenAll`

Very natural for non-blocking async code.

## `Parallel.ForEach`

Classic version is synchronous and blocks until complete.

Example:

```csharp
Parallel.ForEach(items, item => Process(item));
Console.WriteLine("Done");
```

`Done` prints only after all items are processed.

---



# Return values

## `Task.Run`

```csharp
Task<int> task = Task.Run(() => 123);
int result = await task;
```

Returns a value nicely.

---

## `Task.WhenAll`

```csharp
Task<int> t1 = GetAsync1();
Task<int> t2 = GetAsync2();

int[] results = await Task.WhenAll(t1, t2);
```

Returns all results together.

---

## `Parallel.ForEach`

Normally you manage output yourself:

```csharp
var results = new ConcurrentBag<int>();

Parallel.ForEach(items, item =>
{
    results.Add(Process(item));
});
```

Because it is loop-based, not result-array-based like `Task.WhenAll`.

---

# Error handling

## `Task.Run`

If the delegate throws, awaiting the task throws.

## `Task.WhenAll`

If one or more tasks fail, the combined task fails.

## `Parallel.ForEach`

If iterations throw, exceptions are typically wrapped together.

So in all cases, you need error handling, but `Task.WhenAll` is often easiest in async workflows.

---

* **One CPU-heavy operation** -> `Task.Run`
* **Many async I/O operations** -> `Task.WhenAll`
* **Many CPU-heavy items in a collection** -> `Parallel.ForEach`

---

> `Task.Run` is used to queue a piece of work onto the thread pool, usually for CPU-bound work.
> `Task.WhenAll` is used to await multiple tasks together and is especially useful for concurrent async I/O operations.
> `Parallel.ForEach` is used to process a collection in parallel and is typically best for CPU-bound work across many independent items.

---