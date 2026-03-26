---
id: cqrs
topic: CQRS (Command Query Responsibility Segregation)
category: Design & Architecture
---

## Overview

**CQRS (Command Query Responsibility Segregation)** is an architectural pattern that separates operations that **change system state** from operations that **read data**.

In .NET applications:

- **Commands** modify system state and encapsulate business rules.
- **Queries** retrieve data and are optimized for reading.

CQRS can improve:

- clarity of responsibilities
- performance optimization
- scalability

However, it introduces additional complexity and should only be applied when the domain or performance requirements justify it.

---

#  What CQRS Really Means

CQRS separates:

```

Commands → write operations that change system state
Queries  → read operations that return data

````

### Command Rules

Commands:

- Change state
- Usually return minimal data (often nothing or an ID)
- Contain business logic and validation
- Produce side effects

### Query Rules

Queries:

- Read data only
- Must not change system state
- Can be optimized heavily for performance
- Usually return **DTOs**

### Example

```csharp
public record CreateUserCommand(string Email) : IRequest<Guid>;
````

```csharp
public record GetUserByIdQuery(Guid Id) : IRequest<UserDto>;
```

Breaking these rules weakens CQRS separation.

---

#  CQRS Is a Pattern, Not a Framework

CQRS is an **architectural pattern**, not a library or framework.

CQRS does **not automatically imply**:

* Microservices
* Event sourcing
* Separate databases

These technologies are often used together with CQRS but are **not required**.

---

#  MediatR Is Commonly Used in .NET

In .NET, CQRS is commonly implemented using **MediatR**.

CQRS defines the architecture, while MediatR helps enforce it.

MediatR provides:

* decoupling between controllers and business logic
* single-responsibility handlers
* pipeline behaviors (validation, logging, transactions)

### Example Handler

```csharp
public class CreateUserHandler
    : IRequestHandler<CreateUserCommand, Guid>
{
    public async Task<Guid> Handle(
        CreateUserCommand request,
        CancellationToken ct)
    {
        // business logic
    }
}
```

---

#  Validation Belongs to Commands (Not Controllers)

A common CQRS flow:

1. Controller receives request
2. Controller sends command through MediatR
3. Validation executes in pipeline behavior

Example:

```csharp
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
{
    // FluentValidation logic here
}
```

### Benefits

* Controllers remain thin
* Validation is centralized
* Validation logic is reusable

---

#  Read Model ≠ Write Model (Important Concept)

CQRS separates the **write model** and **read model**.

### Write Model

* Domain entities
* Enforces business rules
* Uses aggregates and invariants

### Read Model

* DTOs
* Lightweight projections
* Optimized for queries

Avoid returning domain entities directly in query responses.

Example query projection:

```csharp
var users = await db.Users
    .Select(u => new UserDto
    {
        Id = u.Id,
        Name = u.Name
    })
    .ToListAsync();
```

Reads may use:

* EF Core projections
* Dapper
* SQL views

---

#  CQRS Does Not Require Separate Databases

A common misconception is that CQRS requires separate databases.

In many systems you can use:

* the same database
* the same tables
* different access patterns

CQRS focuses on **logical separation**, not physical separation.

---

### Simple CQRS Architecture

```
Controller
   ↓
Command Handler → EF Core → Database
   ↓
Query Handler → Dapper → Database
```

---

### Advanced CQRS Architecture

```
Write DB
   ↓
Event Bus
   ↓
Read DB (denormalized projections)
```

In advanced architectures, read models may be stored in a separate optimized database.

---

#  Where CQRS Shines

CQRS works well when:

* Write operations contain complex business rules
* Systems are heavily read-oriented
* Read and write workloads differ significantly
* Reporting or dashboards require optimized queries
* Clear domain boundaries improve maintainability

Typical systems using CQRS:

* Order management systems
* Financial systems
* Event-driven systems
* Large enterprise APIs

---

# Example Scenario

Order processing system.

### Commands

* CreateOrder
* CancelOrder
* AddOrderItem

### Queries

* GetOrders
* GetOrderDetails
* GetCustomerOrders

Commands enforce domain rules.

Queries return optimized projections.

---

# Benefits of CQRS

* Clear separation of responsibilities
* Easier unit testing
* Independent read/write optimization
* Improved scalability
* Better domain modeling

---

# Trade-offs

CQRS introduces additional complexity:

* More classes
* More handlers
* Additional infrastructure

CQRS should be avoided when:

* the application is simple CRUD
* domain logic is minimal
* performance requirements are low


---

## Notes

1. **CQRS vs CRUD**

Many systems that use CQRS still expose **CRUD-like APIs externally**.  
The difference lies in the **internal architecture**, not necessarily the API endpoints.

---

2. **DTO Usage**

Queries should return **DTOs instead of EF Core entities**.

Returning domain entities can leak domain logic and persistence concerns into the API layer.

---

3. **Read Optimization**

Queries can be optimized aggressively because they are side-effect free.

Common optimization techniques include:

- SQL views
- denormalized tables
- caching
- read replicas

---

4. **CQRS and Event-Driven Architecture**

CQRS is often combined with event-driven architecture:

Command → Domain Event → Read Model Projection


However, CQRS works perfectly **without event sourcing**.

---
## Domain events

With CQRS, domain events are usually implemented like this:

**Command changes state**
-> **Aggregate/entity raises domain event**
-> **Infrastructure collects and dispatches that event**
-> **Handlers react to it**

That is the common flow.

## 1. Where domain events fit in CQRS

In CQRS:

* **Commands** change data
* **Queries** read data
* **Domain events** describe important business facts that happened during command execution

So a command might be:

* `PlaceOrderCommand`

And the resulting domain event might be:

* `OrderPlacedDomainEvent`

The command says what you want to do.
The domain event says what already happened.

---

## 2. Typical flow

A common CQRS + DDD flow looks like this:

1. API receives `PlaceOrderCommand`
2. Command handler loads aggregate
3. Aggregate method runs business rules
4. Aggregate updates its state
5. Aggregate adds a domain event to its internal event list
6. Unit of Work / DbContext saves changes
7. Domain events are dispatched
8. Event handlers run side effects

Example:

* `Order.Place(...)`
* inside that method:

  * status changes
  * `OrderPlacedDomainEvent` is added

Then later:

* `SendConfirmationEmailHandler` handles `OrderPlacedDomainEvent`
* `ReserveInventoryHandler` handles `OrderPlacedDomainEvent`

The `Order` does not directly call email service or inventory service.

---

## 3. Important idea

A domain event is usually raised by the **write side** only.

That means:

* command side creates and handles domain events
* query side normally does not raise them

Queries should not change state, so they usually do not produce domain events.

---

## 4. Common structure

### Domain event contract

```csharp
public interface IDomainEvent
{
    DateTime OccurredOnUtc { get; }
}
```

If using MediatR:

```csharp
using MediatR;

public interface IDomainEvent : INotification
{
    DateTime OccurredOnUtc { get; }
}
```

---

### Base entity / aggregate root

```csharp
public abstract class Entity
{
    private readonly List<IDomainEvent> _domainEvents = new();

    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}
```

---

### Aggregate raises event

```csharp
public class Order : Entity
{
    public Guid Id { get; private set; }
    public string Status { get; private set; } = "Draft";

    public void Place()
    {
        if (Status != "Draft")
            throw new InvalidOperationException("Only draft orders can be placed.");

        Status = "Placed";

        AddDomainEvent(new OrderPlacedDomainEvent(Id));
    }
}
```

---

### Event class

```csharp
public sealed class OrderPlacedDomainEvent : IDomainEvent
{
    public Guid OrderId { get; }
    public DateTime OccurredOnUtc { get; } = DateTime.UtcNow;

    public OrderPlacedDomainEvent(Guid orderId)
    {
        OrderId = orderId;
    }
}
```

---

## 5. Command handler

The command handler calls the aggregate method. It does not manually create the domain event in most DDD-style designs.

```csharp
public sealed class PlaceOrderCommand : IRequest
{
    public Guid OrderId { get; set; }
}
```

```csharp
public sealed class PlaceOrderCommandHandler : IRequestHandler<PlaceOrderCommand>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IUnitOfWork _unitOfWork;

    public PlaceOrderCommandHandler(
        IOrderRepository orderRepository,
        IUnitOfWork unitOfWork)
    {
        _orderRepository = orderRepository;
        _unitOfWork = unitOfWork;
    }

    public async Task Handle(PlaceOrderCommand request, CancellationToken cancellationToken)
    {
        var order = await _orderRepository.GetByIdAsync(request.OrderId, cancellationToken);

        if (order == null)
            throw new Exception("Order not found.");

        order.Place();

        await _unitOfWork.SaveChangesAsync(cancellationToken);
    }
}
```

Key point:

* handler tells aggregate to do business action
* aggregate raises event
* save operation later dispatches it

---

## 6. How events are dispatched

This is usually done in infrastructure, often around `DbContext.SaveChangesAsync()`.

### Common approach

After saving data:

1. find tracked entities with domain events
2. collect their events
3. clear the events from entities
4. publish them via MediatR or custom dispatcher

Example:

```csharp
public sealed class AppDbContext : DbContext
{
    private readonly IMediator _mediator;

    public AppDbContext(DbContextOptions<AppDbContext> options, IMediator mediator)
        : base(options)
    {
        _mediator = mediator;
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var domainEvents = ChangeTracker
            .Entries<Entity>()
            .Select(x => x.Entity)
            .Where(x => x.DomainEvents.Any())
            .SelectMany(x => x.DomainEvents)
            .ToList();

        var result = await base.SaveChangesAsync(cancellationToken);

        foreach (var entity in ChangeTracker.Entries<Entity>().Select(e => e.Entity))
        {
            entity.ClearDomainEvents();
        }

        foreach (var domainEvent in domainEvents)
        {
            await _mediator.Publish(domainEvent, cancellationToken);
        }

        return result;
    }
}
```

This is one common implementation.

---

## 7. Domain event handlers

Handlers react to domain events.

```csharp
public sealed class SendOrderConfirmationHandler
    : INotificationHandler<OrderPlacedDomainEvent>
{
    public Task Handle(OrderPlacedDomainEvent notification, CancellationToken cancellationToken)
    {
        // send email, write audit log, etc.
        return Task.CompletedTask;
    }
}
```

Another one:

```csharp
public sealed class CreateOrderAuditLogHandler
    : INotificationHandler<OrderPlacedDomainEvent>
{
    public Task Handle(OrderPlacedDomainEvent notification, CancellationToken cancellationToken)
    {
        // store audit record
        return Task.CompletedTask;
    }
}
```

One domain event can have multiple handlers.

That is one of the main benefits.

---

## 8. Why this fits CQRS well

It works well with CQRS because:

* command handler stays focused on one business action
* aggregate keeps business rules
* side effects are separated into event handlers
* adding new reactions later is easier

Without domain events, command handler often becomes bloated:

```csharp
// do order update
// send email
// write audit log
// update loyalty points
// notify warehouse
```

With domain events:

* command handler updates the aggregate
* event handlers do the reactions

That is cleaner.

---

## 9. Domain events vs integration events

This is very important.

### Domain events

* internal to the application
* represent business facts inside the domain
* usually handled in-process

### Integration events

* sent to other services or external systems
* usually published through message broker
* used in distributed systems / microservices

Example:

* `OrderPlacedDomainEvent` -> internal
* `OrderPlacedIntegrationEvent` -> published to RabbitMQ / Service Bus

A common pattern:

1. aggregate raises domain event
2. domain event handler creates integration event
3. integration event is stored/published

Do not treat them as exactly the same thing.

---

## 10. Before save or after save?

There are two common approaches:

### Dispatch before save

* handlers run inside same transaction
* useful if handlers must modify same database transaction

### Dispatch after save

* safer for side effects like email, notifications
* avoids running handlers if main save fails

In many projects, domain events are published **after save** for simplicity.

But if a handler must update other domain objects in the same transaction, some teams dispatch before commit.

A practical rule:

* **same transaction business consistency** -> before commit may make sense
* **side effects / notifications** -> after commit is usually better

---

## 11. Common explanation

> In CQRS, domain events are usually raised by aggregates during command handling. The command handler invokes domain behavior, the aggregate updates its state and records one or more domain events, and the infrastructure layer dispatches those events after persistence. Event handlers then perform side effects or trigger additional business processes without tightly coupling that logic to the command handler or aggregate.

---

## 12. Common mistakes

### Mistake 1: Raising event from handler instead of aggregate

This is possible, but if the event represents a real domain fact, it is usually cleaner for the aggregate to raise it.

### Mistake 2: Putting external service calls inside aggregate

Example: aggregate directly sends email.
That couples domain model to infrastructure.

### Mistake 3: Confusing domain events with integration events

Internal event and cross-service message are not the same concern.

### Mistake 4: Not clearing dispatched events

This can cause duplicate publishing.

### Mistake 5: Using events for everything

Not every field change needs a domain event.
Only meaningful business events.

---

## 13. Practical mental model

Think of it like this:

* **Command** = user asks system to do something
* **Aggregate** = business decision maker
* **Domain event** = record of what happened
* **Event handler** = reactions to that fact
* **Query side** = reads the result later

---

## 14. Very small end-to-end example

```csharp
public sealed class CreateCustomerCommand : IRequest<Guid>
{
    public string Email { get; set; } = default!;
}
```

```csharp
public sealed class CustomerRegisteredDomainEvent : IDomainEvent
{
    public Guid CustomerId { get; }
    public string Email { get; }
    public DateTime OccurredOnUtc { get; } = DateTime.UtcNow;

    public CustomerRegisteredDomainEvent(Guid customerId, string email)
    {
        CustomerId = customerId;
        Email = email;
    }
}
```

```csharp
public class Customer : Entity
{
    public Guid Id { get; private set; }
    public string Email { get; private set; } = default!;

    public static Customer Register(string email)
    {
        var customer = new Customer
        {
            Id = Guid.NewGuid(),
            Email = email
        };

        customer.AddDomainEvent(new CustomerRegisteredDomainEvent(customer.Id, customer.Email));

        return customer;
    }
}
```

```csharp
public sealed class CreateCustomerCommandHandler : IRequestHandler<CreateCustomerCommand, Guid>
{
    private readonly ICustomerRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public CreateCustomerCommandHandler(ICustomerRepository repository, IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Guid> Handle(CreateCustomerCommand request, CancellationToken cancellationToken)
    {
        var customer = Customer.Register(request.Email);

        await _repository.AddAsync(customer, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return customer.Id;
    }
}
```

```csharp
public sealed class SendWelcomeEmailHandler
    : INotificationHandler<CustomerRegisteredDomainEvent>
{
    public Task Handle(CustomerRegisteredDomainEvent notification, CancellationToken cancellationToken)
    {
        // send email
        return Task.CompletedTask;
    }
}
```

That is the basic pattern.

---

## 15. Best simple rule

In CQRS + DDD:

* **command handler** orchestrates
* **aggregate** enforces rules and raises domain events
* **infrastructure** dispatches events
* **handlers** react to them
---






```
