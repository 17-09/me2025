---
title: "CQRS in .NET: when to use it, trade‑offs"
date: 2025-08-28
tags: [".NET", "CQRS", "Architecture", "EF Core", "Dapper"]
description: "Practical CQRS in .NET 8: when it helps, when it hurts, and a full example with EF Core (writes) + Dapper (reads)."
draft: false
---

What is CQRS (in one line)
- Separate the write side (commands that change state) from the read side (queries that return data), so each can optimize independently.

When to use CQRS
- Read/write needs are very different:
  - Reads are hot and need tailored projections, denormalized views, caching, or a different store.
  - Writes need strong validation, invariants, and transactions.
- Performance and scaling:
  - Reads outnumber writes by a lot (10×–100×), and you want to scale/read-optimize without complicating write logic.
- UX wants different shapes than your domain:
  - E.g., “Order with line items and fulfillment status by customer” needs a custom read model that doesn’t match your normalized write schema.
- Complex domains:
  - Clearer boundaries and testing by separating write rules (invariants) from read concerns (reporting, search).
- Event-driven systems:
  - You already publish events and can build read models/projections asynchronously.

When not to use CQRS
- Simple CRUD app, one DB, a few endpoints.
- Team is small, speed matters more than architecture; CQRS will slow you down.
- You can get 90% of the benefit with “read-optimized queries” (e.g., EF AsNoTracking + a couple SQL views).
- You can’t handle the operational overhead (two paths, possibly two stores, projections, eventual consistency).

Pros
- Performance and flexibility on reads without polluting domain logic.
- Cleaner write model with clear invariants and tests.
- Easier to evolve read models for new UX/reporting without risky schema churn.

Cons
- More moving parts (two paths, sometimes two stores).
- Eventual consistency if you use async projections.
- More code to maintain and operate.

A real-world shape
- Orders or bookings: writes enforce business rules (status transitions, inventory holds). Reads need fast lists, filters, totals, dashboards, and customer-friendly shapes.
- In prod I’ve seen:
  - Write: Postgres/MySQL with a normalized schema.
  - Read: denormalized tables or a search index (Elasticsearch/OpenSearch), or tailored SQL views, cached with Redis.
  - Projections updated via outbox + background workers.

Below: a full working example that shows the pattern without too much boilerplate.

What we’ll build
- .NET 8 minimal API
- Writes (commands) via EF Core into SQLite (easy to run)
- Reads (queries) via Dapper with SQL projections
- Endpoints:
  - POST /orders (create)
  - POST /orders/{id}/approve (approve)
  - GET /orders/{id} (query by id)
  - GET /orders (list)

How to run locally
- Create a folder and init a web project:
  - dotnet new web -n CqrsSample && cd CqrsSample
- Add packages:
  - dotnet add package Microsoft.EntityFrameworkCore.Sqlite
  - dotnet add package Dapper
- Replace Program.cs with the code below
- Run:
  - dotnet run
- Try it:
  - Create: curl -X POST http://localhost:8080/orders -H "Content-Type: application/json" -d '{"customerEmail":"a@b.com","items":[{"sku":"SKU-1","quantity":2,"price":19.9},{"sku":"SKU-2","quantity":1,"price":9.5}]}'
  - List: curl http://localhost:8080/orders
  - Approve: curl -X POST http://localhost:8080/orders/{id}/approve
  - Get by id: curl http://localhost:8080/orders/{id}

Full working code (Program.cs)
```csharp
using System.Data;
using Dapper;
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Config
var connString = builder.Configuration.GetConnectionString("App")
                ?? "Data Source=app.db"; // SQLite for a self-contained demo

// EF Core DbContext for writes
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite(connString));

// Dapper connection factory for reads
builder.Services.AddScoped<IDbConnection>(_ => new SqliteConnection(connString));

builder.Services.AddEndpointsApiExplorer();

var app = builder.Build();

// Ensure DB exists (demo). In prod: use migrations + Migrate().
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.EnsureCreated();
}

// Health
app.MapGet("/ping", () => "pong");

// Commands (write side)

// Create Order
app.MapPost("/orders", async (CreateOrderDto dto, AppDbContext db) =>
{
    // Very basic validation; expand as needed
    if (string.IsNullOrWhiteSpace(dto.CustomerEmail))
        return Results.BadRequest("CustomerEmail is required");
    if (dto.Items is null || dto.Items.Count == 0)
        return Results.BadRequest("At least one item is required");
    if (dto.Items.Any(i => i.Quantity <= 0 || i.Price < 0))
        return Results.BadRequest("Invalid item quantity/price");

    var order = Order.Create(dto.CustomerEmail);
    foreach (var item in dto.Items)
        order.AddItem(item.Sku, item.Quantity, item.Price);

    await db.Orders.AddAsync(order);
    await db.SaveChangesAsync();

    return Results.Created($"/orders/{order.Id}", new { order.Id });
});

// Approve Order
app.MapPost("/orders/{id:guid}/approve", async (Guid id, AppDbContext db) =>
{
    var order = await db.Orders.Include(o => o.Items).FirstOrDefaultAsync(o => o.Id == id);
    if (order is null) return Results.NotFound();

    try
    {
        order.Approve();
        await db.SaveChangesAsync();
        return Results.Accepted($"/orders/{order.Id}");
    }
    catch (InvalidOperationException ex)
    {
        return Results.Conflict(new { message = ex.Message });
    }
});

// Queries (read side) using Dapper

// Get by Id
app.MapGet("/orders/{id:guid}", async (Guid id, IDbConnection conn) =>
{
    const string sql = @"
select 
  o.Id, o.CustomerEmail, o.Status, o.CreatedAt,
  i.Id as ItemId, i.Sku, i.Quantity, i.Price
from Orders o
left join OrderItems i on i.OrderId = o.Id
where o.Id = @Id
order by i.Id";

    var rows = await conn.QueryAsync<dynamic>(sql, new { Id = id });

    var first = rows.FirstOrDefault();
    if (first is null) return Results.NotFound();

    var result = new OrderReadDto
    {
        Id = first.ID,
        CustomerEmail = first.CUSTOMEREMAIL,
        Status = first.STATUS,
        CreatedAt = first.CREATEDAT,
        Items = rows
            .Where(r => r.ITEMID != null)
            .Select(r => new OrderItemReadDto
            {
                Id = r.ITEMID,
                Sku = r.SKU,
                Quantity = (int)r.QUANTITY,
                Price = (decimal)r.PRICE
            })
            .ToList()
    };

    return Results.Ok(result);
});

// List (latest N)
app.MapGet("/orders", async (int take, int skip, IDbConnection conn) =>
{
    take = take <= 0 || take > 100 ? 20 : take;
    skip = skip < 0 ? 0 : skip;

    const string sql = @"
select o.Id, o.CustomerEmail, o.Status, o.CreatedAt
from Orders o
order by o.CreatedAt desc
limit @Take offset @Skip";

    var orders = (await conn.QueryAsync<OrderListItemDto>(sql, new { Take = take, Skip = skip })).ToList();
    return Results.Ok(orders);
});

// Run
app.Urls.Add("http://0.0.0.0:8080");
app.Run();


// -------------------- Domain + Data --------------------

public sealed class Order
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public string CustomerEmail { get; private set; } = default!;
    public string Status { get; private set; } = OrderStatus.Pending;
    public DateTime CreatedAt { get; private set; } = DateTime.UtcNow;
    public List<OrderItem> Items { get; private set; } = new();

    private Order() { } // EF

    public static Order Create(string customerEmail)
    {
        if (string.IsNullOrWhiteSpace(customerEmail))
            throw new ArgumentException("CustomerEmail required", nameof(customerEmail));

        return new Order { CustomerEmail = customerEmail.Trim() };
    }

    public void AddItem(string sku, int quantity, decimal price)
    {
        if (string.IsNullOrWhiteSpace(sku)) throw new ArgumentException("SKU required", nameof(sku));
        if (quantity <= 0) throw new ArgumentException("Quantity must be > 0", nameof(quantity));
        if (price < 0) throw new ArgumentException("Price must be >= 0", nameof(price));

        Items.Add(new OrderItem { Sku = sku.Trim(), Quantity = quantity, Price = price });
    }

    public void Approve()
    {
        if (Status == OrderStatus.Approved) throw new InvalidOperationException("Order already approved");
        if (Status != OrderStatus.Pending) throw new InvalidOperationException($"Cannot approve from status {Status}");
        if (Items.Count == 0) throw new InvalidOperationException("Cannot approve order without items");

        Status = OrderStatus.Approved;
    }
}

public static class OrderStatus
{
    public const string Pending = "Pending";
    public const string Approved = "Approved";
}

public sealed class OrderItem
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Sku { get; set; } = default!;
    public int Quantity { get; set; }
    public decimal Price { get; set; }

    public Guid OrderId { get; set; }
    public Order Order { get; set; } = default!;
}

public sealed class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Order>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.CustomerEmail).IsRequired();
            e.Property(x => x.Status).IsRequired();
            e.Property(x => x.CreatedAt).HasConversion(
                v => v, v => DateTime.SpecifyKind(v, DateTimeKind.Utc)); // keep UTC
            e.HasMany(x => x.Items)
             .WithOne(i => i.Order)
             .HasForeignKey(i => i.OrderId)
             .OnDelete(DeleteBehavior.Cascade);
            e.ToTable("Orders");
        });

        b.Entity<OrderItem>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.Sku).IsRequired();
            e.Property(x => x.Quantity).IsRequired();
            e.Property(x => x.Price).IsRequired();
            e.ToTable("OrderItems");
        });
    }
}

// -------------------- DTOs --------------------

public sealed record CreateOrderDto(string CustomerEmail, List<CreateOrderItemDto> Items);
public sealed record CreateOrderItemDto(string Sku, int Quantity, decimal Price);

public sealed class OrderReadDto
{
    public Guid Id { get; set; }
    public string CustomerEmail { get; set; } = default!;
    public string Status { get; set; } = default!;
    public DateTime CreatedAt { get; set; }
    public List<OrderItemReadDto> Items { get; set; } = new();
}

public sealed class OrderItemReadDto
{
    public Guid Id { get; set; }
    public string Sku { get; set; } = default!;
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

public sealed class OrderListItemDto
{
    public Guid Id { get; set; }
    public string CustomerEmail { get; set; } = default!;
    public string Status { get; set; } = default!;
    public DateTime CreatedAt { get; set; }
}
```

Notes and how this maps to CQRS
- Commands (write path): POST /orders and POST /orders/{id}/approve use EF Core to modify state and enforce invariants (domain rules).
- Queries (read path): GET endpoints use Dapper with SQL tailored to what the UI needs. This is where you’d denormalize, join, and shape data for performance.
- Same DB for demo simplicity. In production:
  - Keep writes on your primary (e.g., Postgres on RDS).
  - Build read models via views/materialized views, replicas, or a separate store (e.g., OpenSearch).
  - If you go async projections, use the Outbox pattern to update the read model reliably.

Hardening suggestions (prod)
- Validation: FluentValidation or custom validators on commands.
- Migrations: add EF migrations and run db.Database.Migrate() on startup.
- Transactions: wrap multi‑aggregate writes in a transaction scope.
- Observability: add structured logging + OTEL traces on command and query handlers.
- Security: authN/authZ, row‑level access if multi‑tenant.
- Idempotency: for create/update commands if clients may retry.

If you want, I can split this into a clean multi‑project repo (Domain, Application, Infrastructure, Api) and wire MediatR + FluentValidation + test projects. We can also switch the read side to Postgres + Dapper or views to match your RDS setup.
```