---
title: "Event Sourcing in .NET"
date: 2025-08-28
tags: [".NET", "Event Sourcing", "EF Core", "Architecture", "CQRS"]
description: "Practical Event Sourcing in .NET 8: when it helps, when it hurts, and a full example with an EF Core event store and read model projection."
draft: false
---

Event Sourcing in one line
- Store every state change as an event in an append-only log; rebuild state by replaying events and serve reads from projections.

When to use Event Sourcing
- You need a full audit/history of changes (who did what, when).
- Complex domain invariants benefit from explicit events and replay.
- You want temporal queries (state as of time T) or easy rebuilds of new read models.
- Distributed workflows where event streams are the source of truth.
- You already embrace CQRS and projections (read models).

When not to use it
- Simple CRUD with low domain complexity—use a relational model and call it a day.
- Team lacks bandwidth for the extra operational surface (projections, replays, tooling).
- Strong consistency on reads is mandatory everywhere (eventual consistency is normal here).
- You don’t need audit/history beyond typical DB logs.

Pros
- Perfect audit trail and temporal queries.
- Easy to add new read models later (just replay).
- Clear domain language via events.
- Testing often gets simpler (events in, events out).

Cons
- More moving parts: event store, projections, replays, versioning events.
- Eventual consistency for read models (unless you update synchronously).
- Refactoring old events is hard—evolve with new event types instead.
- Tooling/operational maturity required (backfills, DLQs, rebuilds).

A real-world shape
- Orders/payments: stream per order/payment, events like Created, Authorized, Captured, Refunded. Read models: customer views, finance reports, search indexes.
- HR/ATS (fits your background): stream per candidate or job posting, events like Applied, Screened, Scored, Advanced, Rejected. Read models: recruiter dashboard, funnel metrics.

Working example: Accounts with deposits/withdrawals
- .NET 8 minimal API
- EF Core (SQLite) as a simple event store
- Append-only Events table with optimistic concurrency (unique streamId+version)
- Synchronous projection table AccountsRead for fast reads (atomic updates in the same transaction)
- Endpoints:
  - POST /accounts (open)
  - POST /accounts/{id}/deposit
  - POST /accounts/{id}/withdraw
  - GET /accounts/{id} (read model)
  - GET /accounts/{id}/events (raw stream)
  - GET /accounts (list read model)

How to run
- dotnet new web -n EventSourcingDemo && cd EventSourcingDemo
- dotnet add package Microsoft.EntityFrameworkCore.Sqlite
- Replace Program.cs with the code below
- dotnet run
- Try:
  - Open: curl -s -X POST http://localhost:8080/accounts -H "Content-Type: application/json" -d '{"customerEmail":"a@b.com","openingBalance":100}'
  - Deposit: curl -s -X POST http://localhost:8080/accounts/{id}/deposit -H "Content-Type: application/json" -d '{"amount":50}'
  - Withdraw: curl -s -X POST http://localhost:8080/accounts/{id}/withdraw -H "Content-Type: application/json" -d '{"amount":30}'
  - Get: curl -s http://localhost:8080/accounts/{id}
  - Events: curl -s http://localhost:8080/accounts/{id}/events
  - List: curl -s http://localhost:8080/accounts

Full working code (Program.cs)
```csharp
using System.Text.Json;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Config (SQLite for demo)
var connString = builder.Configuration.GetConnectionString("App") ?? "Data Source=app.db";
builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlite(connString));

var app = builder.Build();

// Create DB (demo). In prod: migrations + Migrate().
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.EnsureCreated();
}

app.MapGet("/ping", () => "pong");

// -------------------- Commands (append events) --------------------

app.MapPost("/accounts", async (OpenAccountDto dto, AppDbContext db) =>
{
    if (string.IsNullOrWhiteSpace(dto.CustomerEmail))
        return Results.BadRequest("customerEmail required");
    if (dto.OpeningBalance < 0)
        return Results.BadRequest("openingBalance must be >= 0");

    var streamId = Guid.NewGuid();
    var now = DateTime.UtcNow;

    await using var tx = await db.Database.BeginTransactionAsync();

    // Check current version (new stream, so should be 0)
    var verRow = await db.MaxVersionRows
        .FromSqlInterpolated($@"select coalesce(max(Version), 0) as MaxVersion from Events where StreamId = {streamId}")
        .AsNoTracking()
        .FirstAsync();

    var currentVersion = verRow.MaxVersion; // 0

    // Build event
    var ev = new EventEntity
    {
        Id = Guid.NewGuid(),
        StreamId = streamId,
        StreamType = "Account",
        Version = currentVersion + 1,
        Name = EventNames.AccountOpened,
        Data = JsonSerializer.Serialize(new AccountOpened { CustomerEmail = dto.CustomerEmail.Trim(), OpeningBalance = dto.OpeningBalance }),
        Metadata = "{}",
        CreatedAt = now
    };

    await db.Events.AddAsync(ev);

    // Upsert projection
    var proj = new AccountRead
    {
        Id = streamId,
        CustomerEmail = dto.CustomerEmail.Trim(),
        Balance = dto.OpeningBalance,
        Version = ev.Version,
        UpdatedAt = now
    };
    await db.AccountsRead.AddAsync(proj);

    await db.SaveChangesAsync();
    await tx.CommitAsync();

    return Results.Created($"/accounts/{streamId}", new { id = streamId });
});

app.MapPost("/accounts/{id:guid}/deposit", async (Guid id, MoneyDto dto, AppDbContext db) =>
{
    if (dto.Amount <= 0) return Results.BadRequest("amount must be > 0");

    var now = DateTime.UtcNow;
    await using var tx = await db.Database.BeginTransactionAsync();

    // Load projection to get current version/balance
    var proj = await db.AccountsRead.FindAsync(id);
    if (proj is null) return Results.NotFound();

    // Next version
    var nextVersion = proj.Version + 1;

    var ev = new EventEntity
    {
        Id = Guid.NewGuid(),
        StreamId = id,
        StreamType = "Account",
        Version = nextVersion,
        Name = EventNames.MoneyDeposited,
        Data = JsonSerializer.Serialize(new MoneyDeposited { Amount = dto.Amount }),
        Metadata = "{}",
        CreatedAt = now
    };
    await db.Events.AddAsync(ev);

    // Update projection
    proj.Balance += dto.Amount;
    proj.Version = nextVersion;
    proj.UpdatedAt = now;

    await db.SaveChangesAsync();
    await tx.CommitAsync();

    return Results.Accepted($"/accounts/{id}");
});

app.MapPost("/accounts/{id:guid}/withdraw", async (Guid id, MoneyDto dto, AppDbContext db) =>
{
    if (dto.Amount <= 0) return Results.BadRequest("amount must be > 0");

    var now = DateTime.UtcNow;
    await using var tx = await db.Database.BeginTransactionAsync();

    var proj = await db.AccountsRead.FindAsync(id);
    if (proj is null) return Results.NotFound();

    if (proj.Balance < dto.Amount)
        return Results.Conflict(new { message = "Insufficient funds" });

    var nextVersion = proj.Version + 1;

    var ev = new EventEntity
    {
        Id = Guid.NewGuid(),
        StreamId = id,
        StreamType = "Account",
        Version = nextVersion,
        Name = EventNames.MoneyWithdrawn,
        Data = JsonSerializer.Serialize(new MoneyWithdrawn { Amount = dto.Amount }),
        Metadata = "{}",
        CreatedAt = now
    };
    await db.Events.AddAsync(ev);

    proj.Balance -= dto.Amount;
    proj.Version = nextVersion;
    proj.UpdatedAt = now;

    await db.SaveChangesAsync();
    await tx.CommitAsync();

    return Results.Accepted($"/accounts/{id}");
});

// -------------------- Queries (read models and streams) --------------------

app.MapGet("/accounts/{id:guid}", async (Guid id, AppDbContext db) =>
{
    var proj = await db.AccountsRead.FindAsync(id);
    if (proj is null) return Results.NotFound();

    var dto = new AccountReadDto
    {
        Id = proj.Id,
        CustomerEmail = proj.CustomerEmail,
        Balance = proj.Balance,
        Version = proj.Version,
        UpdatedAt = proj.UpdatedAt
    };
    return Results.Ok(dto);
});

app.MapGet("/accounts/{id:guid}/events", async (Guid id, AppDbContext db) =>
{
    var events = await db.EventRows
        .FromSqlInterpolated($@"
            select Id, StreamId, StreamType, Version, Name, Data, CreatedAt
            from Events
            where StreamId = {id}
            order by Version asc")
        .AsNoTracking()
        .ToListAsync();

    // Return raw stream (no LINQ shaping)
    var list = new List<object>(events.Count);
    for (int i = 0; i < events.Count; i++)
    {
        var e = events[i];
        list.Add(new
        {
            e.Id,
            e.StreamId,
            e.StreamType,
            e.Version,
            e.Name,
            Data = JsonSerializer.Deserialize<object>(e.Data),
            e.CreatedAt
        });
    }
    return Results.Ok(list);
});

app.MapGet("/accounts", async (AppDbContext db) =>
{
    var rows = await db.AccountListRows
        .FromSqlInterpolated($@"
            select Id, CustomerEmail, Balance, Version, UpdatedAt
            from AccountsRead
            order by UpdatedAt desc
            limit 50")
        .AsNoTracking()
        .ToListAsync();

    var list = new List<AccountReadDto>(rows.Count);
    for (int i = 0; i < rows.Count; i++)
    {
        var r = rows[i];
        list.Add(new AccountReadDto
        {
            Id = r.Id,
            CustomerEmail = r.CustomerEmail,
            Balance = r.Balance,
            Version = r.Version,
            UpdatedAt = r.UpdatedAt
        });
    }
    return Results.Ok(list);
});

app.Urls.Add("http://0.0.0.0:8080");
app.Run();

// -------------------- Domain events (typed) --------------------

static class EventNames
{
    public const string AccountOpened   = "AccountOpened";
    public const string MoneyDeposited  = "MoneyDeposited";
    public const string MoneyWithdrawn  = "MoneyWithdrawn";
}

public sealed record AccountOpened
{
    public string CustomerEmail { get; init; } = default!;
    public decimal OpeningBalance { get; init; }
}

public sealed record MoneyDeposited { public decimal Amount { get; init; } }
public sealed record MoneyWithdrawn { public decimal Amount { get; init; } }

// -------------------- DTOs --------------------

public sealed record OpenAccountDto(string CustomerEmail, decimal OpeningBalance);
public sealed record MoneyDto(decimal Amount);

public sealed class AccountReadDto
{
    public Guid Id { get; set; }
    public string CustomerEmail { get; set; } = default!;
    public decimal Balance { get; set; }
    public int Version { get; set; }
    public DateTime UpdatedAt { get; set; }
}

// -------------------- EF Core --------------------

public sealed class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // Event store and projection
    public DbSet<EventEntity> Events => Set<EventEntity>();
    public DbSet<AccountRead> AccountsRead => Set<AccountRead>();

    // Keyless rows for queries
    public DbSet<EventRow> EventRows => Set<EventRow>();
    public DbSet<AccountListRow> AccountListRows => Set<AccountListRow>();
    public DbSet<MaxVersionRow> MaxVersionRows => Set<MaxVersionRow>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<EventEntity>(e =>
        {
            e.HasKey(x => x.Id);
            e.HasIndex(x => new { x.StreamId, x.Version }).IsUnique(); // optimistic concurrency
            e.Property(x => x.StreamType).IsRequired();
            e.Property(x => x.Version).IsRequired();
            e.Property(x => x.Name).IsRequired();
            e.Property(x => x.Data).IsRequired();
            e.Property(x => x.Metadata).IsRequired();
            e.Property(x => x.CreatedAt).IsRequired();
            e.ToTable("Events");
        });

        b.Entity<AccountRead>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.CustomerEmail).IsRequired();
            e.Property(x => x.Balance).IsRequired();
            e.Property(x => x.Version).IsRequired();
            e.Property(x => x.UpdatedAt).IsRequired();
            e.ToTable("AccountsRead");
        });

        // Keyless query rows
        b.Entity<EventRow>(e => { e.HasNoKey(); e.ToView(null); });
        b.Entity<AccountListRow>(e => { e.HasNoKey(); e.ToView(null); });
        b.Entity<MaxVersionRow>(e => { e.HasNoKey(); e.ToView(null); });
    }
}

// Event store row
public sealed class EventEntity
{
    public Guid Id { get; set; }
    public Guid StreamId { get; set; }
    public string StreamType { get; set; } = default!;
    public int Version { get; set; }
    public string Name { get; set; } = default!;
    public string Data { get; set; } = default!;
    public string Metadata { get; set; } = "{}";
    public DateTime CreatedAt { get; set; }
}

// Projection row
public sealed class AccountRead
{
    public Guid Id { get; set; }
    public string CustomerEmail { get; set; } = default!;
    public decimal Balance { get; set; }
    public int Version { get; set; }
    public DateTime UpdatedAt { get; set; }
}

// Keyless rows for queries
public sealed class EventRow
{
    public Guid Id { get; set; }
    public Guid StreamId { get; set; }
    public string StreamType { get; set; } = default!;
    public int Version { get; set; }
    public string Name { get; set; } = default!;
    public string Data { get; set; } = default!;
    public DateTime CreatedAt { get; set; }
}

public sealed class AccountListRow
{
    public Guid Id { get; set; }
    public string CustomerEmail { get; set; } = default!;
    public decimal Balance { get; set; }
    public int Version { get; set; }
    public DateTime UpdatedAt { get; set; }
}

public sealed class MaxVersionRow
{
    public int MaxVersion { get; set; }
}Here’s a senior, attractive, and straightforward post about **Event Sourcing** for your Hugo blog, with .NET code and crisp advice:

---
title: Event Sourcing in .NET – Simple Guide, Use Cases, and Production Tips  
date: 2024-06-14  
author: Nguyen Thanh Tuyen  
tags: [".NET", "architecture", "event sourcing", "backend"]  
summary: "Unlock the power of Event Sourcing in your .NET apps. Learn the core idea, practical example, when to use it, pros & cons, and what’s critical for production."  
---

## Event Sourcing: What is It, Really?

Imagine you’re building a simple bank account service. Traditionally, you’d store the **current balance**. With **Event Sourcing**, you store every **action (event)** as it happens—like “Deposited $100”, “Withdrew $20”—and build the latest state by replaying these events.

Instead of this record:

| AccountId | Balance |
|-----------|---------|
| 1         | $80     |

You have:

| AccountId | EventType | Amount | Timestamp         |
|-----------|-----------|--------|-------------------|
| 1         | Deposit   | 100    | 2024-06-09 10:00  |
| 1         | Withdraw  | 20     | 2024-06-09 10:05  |

**Current balance** = last known state after replaying all events.

---

## Simple .NET Example

Let’s model this in clean C#:

```csharp
// Define events
public abstract record BankEvent(DateTime Timestamp);
public record Deposited(decimal Amount, DateTime Timestamp) : BankEvent(Timestamp);
public record Withdrew(decimal Amount, DateTime Timestamp) : BankEvent(Timestamp);

public class BankAccount
{
    public List<BankEvent> Events { get; init; } = new();
    public decimal GetBalance()
    {
        decimal balance = 0;
        foreach (var @event in Events)
        {
            switch (@event)
            {
                case Deposited d:  balance += d.Amount; break;
                case Withdrew w:   balance -= w.Amount; break;
            }
        }
        return balance;
    }

    public void Deposit(decimal amount)   => Events.Add(new Deposited(amount, DateTime.UtcNow));
    public void Withdraw(decimal amount)  => Events.Add(new Withdrew(amount, DateTime.UtcNow));
}

// --- Usage ---
var account = new BankAccount();
account.Deposit(100);
account.Withdraw(20);
Console.WriteLine($"Balance: {account.GetBalance()}"); // 80
```

---

## When (and When Not) Should You Use Event Sourcing?

### Best fit when:
- You need a full audit trail for every change (banking, bookings)
- You must reconstruct past states or debug tricky issues
- Your business logic is driven by events or workflows

### Avoid when:
- Your domain is simple (CRUD, no special audit/history need)
- You have HIGH event throughput and don’t need to replay, causing performance issues
- Regulatory pressure forbids storing raw actions/events

---

## Pros & Cons

✔ **Pros:**
- Complete, immutable history of all changes (perfect for audit)
- Can “replay” history to rebuild state (great for debugging/migrations)
- Enables CQRS (separate write/read models)

✖ **Cons:**
- Harder to query current state (must aggregate events)
- Data “explodes” over time—storage and snapshotting required
- Event schema evolution gets tricky (must version, migrate events)

---

## Production Readiness: Key Considerations

- **Event Store:** Use mature solutions (EventStoreDB, or append-only tables in PostgreSQL)
- **Snapshots:** Store periodic snapshots to avoid replaying 1M+ events for each aggregate
- **Versioning:** Every event schema change must be backward compatible, or you need migration logic
- **Replay Safety:** Events must be idempotent—can be processed more than once without harm
- **Performance:** Tune for write-heavy workloads, leverage batching, and efficient read models (CQRS)

---

## Final Thoughts

**Event Sourcing** unlocks powerful patterns beyond simple CRUD, but comes at a cognitive and operational cost. Use it where the trade-off is justified—think audit, workflows, or anytime you need to “time travel” your data.

Stay pragmatic. Not every system needs it, but when you do, .NET makes it approachable and robust.

---

Ready to architect your next event-driven service? Questions? Comments below!

---

*Nguyen Thanh Tuyen – Leading scalable backend and cloud architectures for fast-growing teams*