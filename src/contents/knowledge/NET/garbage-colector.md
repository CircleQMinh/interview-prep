---
id: garbage-colector
topic: Garbage Collector
category: .NET
---

## Simple idea

When your C# program creates objects, it is like putting things into rooms in a building:

* a customer object
* an order object
* a list
* a string

These things take up space in memory.

Instead of making you manually clean up every unused object, .NET has a **garbage collector (GC)** whose job is:

> Find objects nobody is using anymore, then remove them to free space.


## What “still being used” means

An object stays alive if the program can still reach it.

For non-technical wording:

* If your program still has a way to find that object, it stays
* If your program has no way to find it anymore, it becomes garbage

Example:

You write a function that creates an object:

```csharp
void DoWork()
{
    var user = new User();
}
```

When the function ends, the variable `user` disappears.

Now, if nothing else is pointing to that `User` object, it is like:

* nobody knows where it is
* nobody can use it anymore

So GC marks it as something that can be cleaned later.

---


## What GC does step by step

A friendly version:

### 1. Your program creates objects

For example:

* user
* cart
* product list
* message

These are placed in memory.

### 2. Some objects stop being used

Maybe a function ends, or a value is replaced, and nothing refers to the old object anymore.

### 3. GC checks what is still reachable

It asks:

> “Can the program still get to this object?”

If yes, keep it.
If no, it is garbage.

### 4. GC removes unused objects

That frees memory so the program can keep running without wasting space.

---

## Why .NET does this

Without GC, developers would need to manually free memory all the time.

That is hard and error-prone.

People might:

* forget to clean memory
* clean it too early
* clean the wrong thing

GC helps prevent that.

So in simple words:

> GC is an automatic memory cleanup system.

---

## Why objects are not deleted immediately

Because stopping the whole program to clean every second would be inefficient.

So GC waits until it makes sense.

Like a cleaner in an office:

* not every minute
* not after every paper is discarded
* but when the room starts getting messy enough

---