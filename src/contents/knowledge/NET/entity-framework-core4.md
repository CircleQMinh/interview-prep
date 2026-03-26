---
id: entity-framework-core4
topic: Entity Framework Core (EF Core) Part 4
category: .NET
---

### Is EF Core thread-safe?

`DbContext` is not thread-safe and must not be used concurrently.

### How does it handle connections?

EF Core generally opens a connection only for the current operation and closes it immediately after, while the database driver handles connection pooling underneath.

DbContext in EF Core is not thread-safe. It is designed for a single unit of work and should not be used concurrently from multiple threads. You should await each async database call before using the same context again, or create separate contexts for parallel work. During runtime, EF Core usually opens the database connection right before executing a query or save operation and closes it immediately afterward. EF Core itself does not implement connection pooling; it relies on the underlying ADO.NET provider, which usually pools connections by default.

## Common mistakes that cause threading issues

### Reusing the same context across parallel tasks

Bad:

```csharp
var usersTask = _db.Users.ToListAsync();
var productsTask = _db.Products.ToListAsync();
await Task.WhenAll(usersTask, productsTask);
```

### Forgetting to await before reusing the context

Bad:

```csharp
var usersTask = _db.Users.ToListAsync();
// do something else that uses _db before await
var orders = await _db.Orders.ToListAsync();
```

##  How to use it correctly

### In a normal web app

Typical pattern:

* HTTP request starts
* DI creates one scoped `DbContext`
* your service/controller uses it
* `SaveChangesAsync()` runs if needed
* request ends
* context is disposed. ([Microsoft Learn][1])

That is the normal and recommended EF Core lifecycle.

### If you need parallel database work

Use **separate `DbContext` instances**, not one shared instance.

For example:

* one context per parallel worker
* or use `IDbContextFactory<TContext>` to create separate contexts when needed

The key point is not “parallel is forbidden”; the key point is:

> Parallel work must not share the same `DbContext`.

## How EF Core handles connections at runtime 

EF Core usually borrows a database connection from the provider’s connection pool when it needs to execute a query or save changes, and then closes it so the connection is returned to the pool afterward.

Think of the pool like a box of reusable DB connections.

When EF Core needs one:

* it asks the provider to open a connection
* if the pool already has an idle one, that one is reused
* otherwise a new physical connection may be created
* when the work is done, the connection is returned to the box for reuse 

## More accurate flow

Think of it like this:

1. Your app has a `DbContext`
2. EF Core needs to run a query
3. EF Core asks the provider for a DB connection
4. The provider may give it an existing pooled connection
5. EF Core runs the SQL
6. EF Core closes the connection
7. The provider returns that connection to the pool for reuse 