---
id: azure-function
topic: Azure Function
category: Azure
---



# Azure Functions

## What is Azure Functions?

Azure Functions is a serverless compute service that allows you to run code triggered by events without managing servers.

### Key Characteristics

* Event-driven
* Auto-scaling
* Pay-per-execution
* Managed infrastructure

### Common Use Cases

* Background jobs
* File processing
* Integrations
* APIs
* Message processing

---

## What is Serverless?

Serverless means:

* No server management
* Automatic scaling
* Billing based on execution time

You still write backend code — Azure manages the runtime and scaling.

---

## Cold Start

Cold start is the delay that occurs when Azure starts a new function instance after inactivity.

Occurs primarily in:

* Consumption plan

### Solutions

* Premium plan
* Keep-alive ping
* Warm-up trigger

---

## In-Process vs Isolated Worker Model

### In-Process

* Runs inside the Functions host
* Older model
* Less flexible

### Isolated Worker (Recommended for .NET 8+)

* Runs in a separate process
* Full control over .NET features (DI, config, logging)
* Decoupled from host runtime
* Uses `Microsoft.Azure.Functions.Worker` SDK

### Typical Setup

* Create `Program.cs`
* Call `ConfigureFunctionsWorkerDefaults()`
* Register services via `builder.Services`
* Use constructor injection

---

## Function Timeout

### Consumption Plan

* Default: 5 minutes
* Maximum: 10 minutes

```json
{
  "functionTimeout": "00:05:00"
}
```

Can increase to:

```json
{
  "functionTimeout": "00:10:00"
}
```

Cannot exceed 10 minutes.

---

### Premium Plan

* Default: 30 minutes
* Maximum: Practically configurable (no fixed hard limit)


## Durable Functions

Durable Functions is an Azure Functions extension for building stateful, reliable workflows in a serverless environment. It lets you coordinate activity functions using an orchestrator function, with built-in support for persistence, checkpoints, retries, timers, external events, and long-running workflows. It is useful for multi-step business processes, approvals, batch jobs, and any workflow that needs to survive restarts and resume later.

Normal Azure Function:

```text
trigger → do work → finish
```

Durable Function:

```text
trigger → start workflow → run steps → wait → resume later → finish
```

That makes it useful for workflows like:

* order processing
* approval flows
* file-processing pipelines
* background jobs with retries
* long-running multi-step business processes.

## Core pieces of Durable Functions

### 1. Orchestrator function

This is the **workflow coordinator**.

It decides:

* what step runs first
* what runs in parallel
* what to do after a result comes back
* when to wait
* when to retry
* when the workflow is done.

Example idea:

```text
SubmitOrder
  → validate order
  → charge payment
  → reserve inventory
  → send confirmation
```

### 2. Activity function

This is where the **actual work** happens.

Activities are used for:

* database access
* HTTP calls
* sending emails
* CPU work
* calling other services

In practice:

* orchestrator = workflow logic
* activity = real side-effect work. 

### 3. Durable entity

This is a small **durable state object** with operations.

You can use entities for things like:

* counters
* locks
* shopping-cart-like state
* per-user/per-item small state

Entities manage explicit state and are different from orchestrators, which represent state through workflow progress. 



## 1. Function chaining

Use this when steps must run **in order**, and each step may depend on the previous one’s output. Microsoft describes function chaining as running a sequence of activities where the output of one is often passed to the next. ([Microsoft Learn][2])

### Example scenario

Order workflow:

```text
Validate order → Charge payment → Reserve inventory → Send email
```

### Orchestrator

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask;
using Microsoft.DurableTask.Client;

public static class OrderWorkflow
{
    [Function("OrderWorkflow")]
    public static async Task<string> Run(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        var order = context.GetInput<OrderRequest>()!;

        var validated = await context.CallActivityAsync<bool>("ValidateOrder", order);
        if (!validated)
        {
            return "Order invalid";
        }

        var paymentResult = await context.CallActivityAsync<string>("ChargePayment", order);
        var inventoryResult = await context.CallActivityAsync<string>("ReserveInventory", order);
        await context.CallActivityAsync("SendConfirmationEmail", order);

        return $"Done: {paymentResult}, {inventoryResult}";
    }
}
```

### Activities

```csharp
public record OrderRequest(int OrderId, decimal Amount, string Email);

public static class OrderActivities
{
    [Function("ValidateOrder")]
    public static bool ValidateOrder([ActivityTrigger] OrderRequest order)
        => order.Amount > 0;

    [Function("ChargePayment")]
    public static string ChargePayment([ActivityTrigger] OrderRequest order)
        => $"Payment charged for order {order.OrderId}";

    [Function("ReserveInventory")]
    public static string ReserveInventory([ActivityTrigger] OrderRequest order)
        => $"Inventory reserved for order {order.OrderId}";

    [Function("SendConfirmationEmail")]
    public static void SendConfirmationEmail([ActivityTrigger] OrderRequest order)
    {
        // send email
    }
}
```

### Why this is chaining

Each activity is awaited in order:

* validate first
* then payment
* then inventory
* then email

That is the simplest Durable Functions pattern.

---

## 2. Fan-out / fan-in

Use this when many items can be processed **in parallel**, then combined. Microsoft describes fan-out/fan-in as running multiple activities in parallel and then aggregating the results. ([Microsoft Learn][3])

### Example scenario

Process 100 uploaded files:

```text
Get file list → process all files in parallel → combine results
```

### Orchestrator

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask;

public static class FileProcessingWorkflow
{
    [Function("FileProcessingWorkflow")]
    public static async Task<FileProcessingSummary> Run(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        var files = context.GetInput<List<string>>()!;

        var tasks = new List<Task<FileResult>>();

        foreach (var file in files)
        {
            tasks.Add(context.CallActivityAsync<FileResult>("ProcessFile", file));
        }

        var results = await Task.WhenAll(tasks);

        var total = results.Length;
        var successCount = results.Count(x => x.Success);

        return new FileProcessingSummary(total, successCount, results.ToList());
    }
}

public record FileResult(string FileName, bool Success);
public record FileProcessingSummary(int Total, int SuccessCount, List<FileResult> Results);
```

### Activity

```csharp
public static class FileActivities
{
    [Function("ProcessFile")]
    public static FileResult ProcessFile([ActivityTrigger] string fileName)
    {
        // pretend to resize image / parse CSV / virus scan / etc.
        return new FileResult(fileName, true);
    }
}
```

### Why this is fan-out/fan-in

* **fan-out**: start many `ProcessFile` activities
* **fan-in**: `Task.WhenAll(tasks)` waits for all results and aggregates them

This is very useful for:

* file processing
* batch jobs
* image/video work
* parallel API calls where order does not matter

---

## 3. Durable entity in action

Durable entities are for **small, addressable pieces of state**, like counters, carts, locks, or per-user state. Microsoft describes entities as operations over explicit state, unlike orchestrators which represent state through workflow progress. ([Microsoft Learn][4])

A classic example is a **Counter** entity.

### Entity

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask.Entities;

public class Counter
{
    public int Value { get; set; }

    public void Add(int amount) => Value += amount;
    public void Reset() => Value = 0;
    public int Get() => Value;

    [Function(nameof(Counter))]
    public static Task Run([EntityTrigger] TaskEntityDispatcher dispatcher)
        => dispatcher.DispatchAsync<Counter>();
}
```

### HTTP starter that signals the entity

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.DurableTask.Client;
using System.Net;

public static class CounterApi
{
    [Function("IncrementCounter")]
    public static async Task<HttpResponseData> IncrementCounter(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = "counter/{id}/add/{amount:int}")]
        HttpRequestData req,
        string id,
        int amount,
        [DurableClient] DurableTaskClient client)
    {
        var entityId = new EntityInstanceId(nameof(Counter), id);

        await client.Entities.SignalEntityAsync(entityId, "Add", amount);

        var response = req.CreateResponse(HttpStatusCode.Accepted);
        await response.WriteStringAsync($"Counter {id} incremented by {amount}");
        return response;
    }

    [Function("GetCounter")]
    public static async Task<HttpResponseData> GetCounter(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = "counter/{id}")]
        HttpRequestData req,
        string id,
        [DurableClient] DurableTaskClient client)
    {
        var entityId = new EntityInstanceId(nameof(Counter), id);

        var state = await client.Entities.GetEntityAsync<Counter>(entityId);

        var response = req.CreateResponse(HttpStatusCode.OK);
        await response.WriteAsJsonAsync(new
        {
            CounterId = id,
            Value = state?.State?.Value ?? 0
        });

        return response;
    }
}
```

### What this does

If you call:

```text
POST /api/counter/user-42/add/5
POST /api/counter/user-42/add/3
GET  /api/counter/user-42
```

you get:

```json
{ "counterId": "user-42", "value": 8 }
```

### When durable entities are useful

Good use cases:

* per-user quota counters
* shopping cart state
* job progress counters
* locks/semaphores
* small aggregates keyed by ID

Not a great fit for:

* large relational datasets
* heavy querying/reporting
* replacing your main database

---

## How to choose between them

### Function chaining

Use when:

* steps must happen in order
* later steps depend on earlier results

Example:

* order processing
* onboarding workflow
* approval pipeline

### Fan-out / fan-in

Use when:

* many independent tasks can run in parallel
* then you need one final aggregate result

Example:

* process many files
* call many services in parallel
* batch imports

### Durable entity

Use when:

* you need small stateful objects identified by key
* you want commands like add/reset/get against that state

Example:

* counters
* carts
* per-tenant state
