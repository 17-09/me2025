---

title: 6 Essential Design Patterns for Microservices 
date: 2024-06-14  
author: Nguyen Thanh Tuyen  
tags: [".NET", "design patterns", "microservices", "architecture"]  
summary: "Timeless design patterns for reliable, scalable microservices."  
---

# 6 Essential Design Patterns for Microservices

Modern distributed systems aren’t just about slapping endpoints together—they’re about ensuring clarity, flexibility, and maintainability as complexity grows. Here are **six key patterns** I use with confidence in .NET microservices, each illustrated with concise code and a real-world scenario.

---

## 1. Builder Pattern

**Concept**  
Build complex objects step by step, especially when they have many configuration options or defaults.

**Code Example:**  
```csharp
public class Email
{
    public string To { get; }
    public string Subject { get; }
    public string Body { get; }

    private Email(string to, string subject, string body)
        => (To, Subject, Body) = (to, subject, body);

    public class Builder
    {
        private string _to, _subject, _body;
        public Builder WithTo(string to) { _to = to; return this; }
        public Builder WithSubject(string subject) { _subject = subject; return this; }
        public Builder WithBody(string body) { _body = body; return this; }
        public Email Build() => new Email(_to, _subject, _body);
    }
}
```
**Usage:**  
```csharp
var email = new Email.Builder()
    .WithTo("alice@site.com")
    .WithSubject("Welcome!")
    .WithBody("Thanks for registering.")
    .Build();
```
**Real-World Use:**  
Apply builder when assembling requests for APIs (email, payments, cloud resources), or constructing DTOs with many optional fields.

---

## 2. Adapter Pattern

**Concept**  
Convert the interface of a legacy/external system into one your app expects—clean, testable, and versioned.

**Code Example:**  
```csharp
public interface IPaymentService
{
    Task Charge(string userId, decimal amount);
}

public class LegacyPaymentAdapter : IPaymentService
{
    private readonly LegacyVendorApi _api = new();
    public Task Charge(string userId, decimal amount)
        => _api.MakePaymentRequest(userId, (double)amount);
}
```
**Real-World Use:**  
Perfect when integrating with external systems or during migrations. Keep your domain untouched by vendor changes or weird APIs.

---

## 3. Mediator Pattern

**Concept**  
Centralize communication between objects—no direct references. Use for CQRS, command, and event flows. Great for scaling large service logic.

**.NET Example with MediatR:**  
```csharp
public record CreateOrderCommand(string ProductId, int Quantity) : IRequest<Guid>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    // Inject dependencies (repositories, etc)
    public Task<Guid> Handle(CreateOrderCommand cmd, CancellationToken ct) {/*...*/}
}
```
**Usage:**  
```csharp
var orderId = await mediator.Send(new CreateOrderCommand("prod-101", 5));
```
**Real-World Use:**  
Use for in-process CQRS, clean request/response logic, or decoupling workflow steps inside your microservice.

---

## 4. Observer Pattern

**Concept**  
Allow components to “subscribe” and react to events without tight coupling—fundamental for event-driven microservices.

**.NET Example:**  
```csharp
// Event
public class UserRegisteredEvent
{
    public string UserId { get; set; }
}

// Publisher
public class UserService
{
    public event Action<UserRegisteredEvent> UserRegistered;

    public void RegisterUser(string userId)
    {
        // ... registration logic ...
        UserRegistered?.Invoke(new UserRegisteredEvent { UserId = userId });
    }
}

// Subscriber
userService.UserRegistered += evt => emailService.SendWelcome(evt.UserId);
```
**Production Tip:**  
In distributed systems, use message brokers (RabbitMQ, Kafka) or .NET buses (`CAP`, `MassTransit`) for cross-service events.

---

## 5. Strategy Pattern

**Concept**  
Dynamically swap logic or algorithms without rewiring code. Enables plug-and-play policies and extensibility.

**Code Example:**  
```csharp
public interface IPricingStrategy { decimal Calculate(decimal basePrice); }

public class RegularPricing : IPricingStrategy
{
    public decimal Calculate(decimal basePrice) => basePrice;
}
public class PremiumPricing : IPricingStrategy
{
    public decimal Calculate(decimal basePrice) => basePrice * 0.85m;
}
public class OrderService
{
    private readonly IPricingStrategy _strategy;
    public OrderService(IPricingStrategy strategy) => _strategy = strategy;
    public decimal GetPrice(decimal price) => _strategy.Calculate(price);
}
```
**Usage:**  
Switch pricing, routing, or discounting algorithms via DI or configuration at runtime.

---

## 6. Command Pattern

**Concept**  
Encapsulate requests (like jobs or actions) as objects, enabling queuing, logging, or retries.

**Code Example:**  
```csharp
public interface ICommand { Task ExecuteAsync(); }
public class SendEmailCommand : ICommand
{
    public Task ExecuteAsync() { /* send email logic */ }
}
public class CommandQueue
{
    public async Task RunAsync(ICommand command) => await command.ExecuteAsync();
}
```
**Real-World Use:**  
Background jobs, message queues, or orchestrated workflows (like CreateUser → SendWelcomeEmail → LogActivity).

---

# Wrapping Up

**Builder, Adapter, Mediator, Observer, Strategy, and Command** patterns aren’t just textbook classics—they’re the backbone of robust and agile microservices. Use them to enforce boundaries, enable testing, and embrace change.

What pattern has changed how you build services? What gave you trouble? Share your insights and questions in the comments!

---

*Nguyen Thanh Tuyen – Technical Lead, building performant, flexible cloud and microservices with discipline and pragmatism.*