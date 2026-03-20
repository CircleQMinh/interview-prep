---
id: microservices
topic: Microservices
category: Design & Architecture
---

## 1. What microservices are

A **microservices architecture** structures an application as a set of **small, autonomous services**, where each service owns a specific business capability, can be deployed independently, and communicates through well-defined interfaces. A key idea is **independent change**: teams should be able to evolve one service without having to redeploy the whole system. Microsoft also highlights that an API gateway is commonly used as the entry point for clients in this style. 

In interviews, it is good to emphasize that microservices are **not just “many APIs”**. They are an architectural approach centered on:

* bounded business capabilities
* team autonomy
* independent deployment
* decentralized data ownership
* explicit contracts between services 

## 2. Why companies use microservices

Microservices are usually adopted to improve:

* **scalability**: scale only the hot services
* **deployability**: release one service without redeploying everything
* **maintainability**: smaller codebases are easier to understand
* **team ownership**: one team can own one domain or service
* **technology flexibility**: different services can use different stacks when justified 

But interviewers also expect you to say the trade-off: microservices add **distributed-system complexity**. Once you split a system, you introduce network calls, partial failures, eventual consistency, observability needs, deployment complexity, and harder testing. AWS explicitly notes that decomposition decisions should balance benefits against complexity. 

## 3. When microservices are a good fit

Microservices are usually a better fit when:

* the system is large or growing
* multiple teams need to work independently
* parts of the system scale differently
* the business domain has clear boundaries
* frequent, independent releases matter 

They are often a poor fit when:

* the product is still small and changing rapidly
* the team is small
* the domain boundaries are unclear
* the organization lacks DevOps, monitoring, and automation maturity

That is a strong interview point: **a modular monolith is often the better starting point**, and services should usually be extracted when there is a real need, not by default. This is consistent with the “evolutionary design” mindset Fowler describes.

---

# 4. Service decomposition

## What it means

**Service decomposition** is the process of deciding **how to split the system into services**. This is one of the most important microservice design questions, because bad boundaries create chatty services, shared data problems, and constant coordination overhead.

## Best principle: decompose by business capability / bounded context
For example:

* **Order Service**
* **Payment Service**
* **Inventory Service**
* **Shipping Service**

is usually better than:

* **Controller Service**
* **Database Service**
* **Validation Service**

Microsoft’s guidance explicitly connects microservice boundaries to **DDD bounded contexts** and **context mapping**. Each bounded context is autonomous, owns a specific domain capability, and exposes integration contracts to others. 

## Common decomposition approaches

### A. Decompose by business capability

Split by what the business does.

Examples:

* Catalog
* Ordering
* Billing
* Identity
* Notification

This is often the safest default because the boundary maps to real business responsibility.

### B. Decompose by bounded context

Use domain-driven design to identify areas where terms, rules, and models differ.

In DDD, the same business term can mean different things in different bounded contexts. For example, Customer in CRM may contain rich relationship and sales information, while Customer in Ordering may only contain data needed to place and fulfill orders. Instead of forcing a single shared model across the whole system, each bounded context should own its own model and business rules. This reduces coupling and makes each service more aligned with its actual responsibility.

Example:

* “Customer” in CRM may be richer than “Customer” in Ordering
* “Product” in Catalog may differ from “Inventory Item” in Warehouse

This avoids forcing one shared model across the whole organization.

### C. Strangler Fig migration

When moving from a monolith, gradually carve out one capability at a time instead of rewriting everything.

Instead of doing this:

* stop the monolith project
* rewrite the whole system
* switch everything over in one big release

you do this instead:

* keep the monolith running
* pick one capability to extract
* build that capability as a new service
* route that part of the traffic to the new service
* repeat until the monolith gradually shrinks


## Good decomposition signs

A service boundary is usually good when:

* it owns a clear business capability
* it has high internal cohesion
* it changes for its own reasons
* it minimizes cross-service calls
* it can own its own data
* one team can own it end-to-end

## Bad decomposition signs

A boundary is probably wrong when:

* every user request touches many services
* services are very “chatty”
* many services share the same database
* changes in one service constantly require changes in others
* service names reflect technical layers instead of business concepts
* transactions require tight synchronous coordination everywhere 

## Database-per-service principle

A very important point:

> Each microservice should own its data.

The **database per service** pattern exists to preserve loose coupling. Other services should not directly read or write another service’s database; they should go through that service’s API or events. 


Why this matters:

* prevents hidden coupling
* allows schema changes per service
* supports independent deployment
* protects domain ownership

---

# 5. Inter-service communication

## What it is

Inter-service communication is how services talk to each other. The main design decision is usually:

* **synchronous communication**
* **asynchronous communication** 

## A. Synchronous communication

Typical technologies:

* HTTP/REST
* gRPC
* direct RPC-style calls 
### Pros

* simple mental model
* immediate response
* easier for request/response use cases
* straightforward for queries and simple orchestration

### Cons

* temporal coupling: caller depends on callee being available now
* more latency
* risk of cascading failures
* long dependency chains can become an anti-pattern



## B. Asynchronous communication

Typical technologies:

* message broker / queue
* publish-subscribe events
* event bus 

Microservices.io lists several async styles:

* request/reply
* notification
* request/asynchronous response
* publish/subscribe 

### Pros

* looser coupling
* better resilience
* better scalability
* services do not need all downstream services available immediately
* supports event-driven architecture

### Cons

* harder debugging
* eventual consistency
* duplicate/out-of-order message handling
* more operational components
* more complex error recovery 

## Synchronous vs asynchronous: when to use which

Use **synchronous** when:

* the caller needs an immediate result
* the interaction is simple
* the dependency is acceptable
* the operation is query-like

Use **asynchronous** when:

* work can happen later
* you want loose coupling
* you need to broadcast a domain event
* workflows cross service boundaries
* resilience and scalability matter more than immediate response 


## Communication best practices

### Minimize inter-service calls

A key design rule from Microsoft’s .NET guidance is: **the fewer communications between microservices, the better**. If services constantly call each other, your boundaries may be wrong. 

### Design for failure

In distributed systems, calls can fail, timeout, retry, or partially succeed. Common resilience patterns include:

* retries = try again
* timeouts = stop waiting after a limit
* circuit breaker = temporarily stop calling a failing dependency
* idempotency = same request/message can be processed more than once without causing wrong side effects
* dead-letter handling for messages  = move failed/unprocessable messages aside for inspection instead of retrying forever

### Prefer events for cross-service propagation

When one service changes important business state, it can publish an event so other services react independently. Microsoft notes that asynchronous message-based communication is especially important for propagating changes across bounded contexts and related domain models. 

### Support observability

Distributed systems need:

* log aggregation
* metrics
* distributed tracing
* health checks
* exception tracking 

# 6. API Gateway

## What it is

An **API Gateway** is a centralized entry point between clients and backend services. It acts as a reverse proxy, routes requests to the right services, and can handle cross-cutting concerns such as authentication, SSL termination, rate limiting, logging, and caching. 

## Why it exists

Without an API gateway, clients may need to call many services directly. Microsoft lists several resulting problems:

* complex client code
* tight client/backend coupling
* more network round trips and latency
* duplicated cross-cutting logic
* protocol limitations
* larger attack surface 

## Benefits of an API gateway

* simpler client integration
* decouples client from internal service layout
* centralized security and policies
* easier versioning and routing
* request aggregation
* better control over public exposure


### Risks

* becomes another component to manage
* can become a bottleneck if badly designed
* may become a single point of failure without HA
* can turn into a “god gateway” with too much business logic
* routing/config changes need discipline and automation 



# 7. Core microservices design principles interviewers expect

These are the principles worth memorizing:

### 1. Single business capability per service

Each service should do one business thing well. 

### 2. Loose coupling, high cohesion

Minimize dependencies between services, maximize internal consistency. 

### 3. Independent deployment

A service should be releasable without coordinating a full-system deployment.

### 4. Own your data

No shared database across services as the default design. 

### 5. Smart endpoints, dumb pipes

* Smart endpoints = the services themselves contain the business rules and decision-making
* Dumb pipes = the communication layer should stay simple, mainly just moving requests/messages from one service to another

### 6. Design for failure

Networks fail. Services timeout. Messages duplicate. Build resilience in from the start. 

### 7. Prefer eventual consistency over distributed locking

Cross-service workflows typically use local transactions plus coordination patterns such as Saga.

### 8. Strong observability

Tracing, logging, metrics, and health checks are mandatory, not optional. 

### 9. Automate infrastructure and deployments

Microservices without CI/CD, automation, and platform maturity become painful very quickly. Microsoft also recommends automation and infrastructure-as-code for managing gateway and routing changes. 

---



# 8. Common interview questions and strong answer points

## “How would you decompose a monolith into microservices?”

Good answer:

* start from business capabilities / bounded contexts
* identify ownership and change boundaries
* extract high-value, high-change, or independently scalable domains first
* keep data ownership with each new service
* use strangler pattern instead of a big-bang rewrite
* monitor service call patterns and refine boundaries over time 

## “Should microservices share a database?”

Best answer:

* generally no
* each service should own its own database
* sharing a DB creates tight coupling and breaks autonomy
* use APIs, events, replicas, or composition patterns instead 

## “When do you choose synchronous vs asynchronous communication?”

Best answer:

* synchronous for immediate request/response needs
* asynchronous for workflows, notifications, domain events, and decoupling
* avoid long synchronous dependency chains 

## “Why use an API gateway?”

Best answer:

* single entry point
* hides internal service topology
* centralizes cross-cutting concerns
* reduces client complexity
* can aggregate backend calls 

## “What are the drawbacks of microservices?”

Best answer:

* operational complexity
* harder testing
* distributed transactions
* more infrastructure
* latency and failure handling
* observability becomes critical 

## “What are signs your microservice design is bad?”

Best answer:

* too many cross-service calls
* shared database
* unclear ownership
* services too small or too coupled
* lots of synchronous chains
* every feature requires touching many services

# 10. What to memorize for interviews

Memorize these high-value points:

* Microservices are about **business capability + autonomy**, not just smaller projects. 
* Decompose by **bounded context/business capability**. 
* Each service should **own its data**.
* Prefer **fewer inter-service calls**; chatty systems are a warning sign. 
* Use **sync** for immediate answers, **async** for decoupling and workflows. 
* API Gateway = **single entry point + routing + cross-cutting concerns + aggregation**. 
* Service mesh is mainly for **internal service-to-service infrastructure concerns**. 
* Microservices solve some problems, but they also create distributed-system problems.