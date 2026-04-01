# Entity Framework – Shared Transaction Between DbContext Instances

Sample project demonstrating how to pass a database transaction from a **parent** `DbContext` to a **child** `DbContext` instance so that both participate in the same atomic operation.

Both classic **Entity Framework 6** and **Entity Framework Core** are covered, each in its own project.

Created as an answer to the Stack Overflow question:
[EF6 connection management](https://learn.microsoft.com/en-us/ef/ef6/fundamentals/connection-management)

---

## Problem

There are scenarios where you need two separate `DbContext` instances to share a single database transaction — for example, when a child service or repository uses its own `DbContext` but must participate in a transaction started by a parent. By default, each `DbContext` manages its own connection and transaction, so you need explicit steps to share one.

---

## Solution Overview

The key is to:
1. Start a transaction on the **parent** `DbContext`.
2. Pass the **same underlying `DbConnection`** to the child `DbContext` so both share the same physical connection.
3. Instruct the child `DbContext` to **use the existing transaction** via `UseTransaction` / `UseTransactionAsync`.
4. Commit or roll back from the parent, affecting all changes made by both contexts.

---

## Entity Framework 6 (`EntityFramework.SharedTransaction`)

### How it works

- The child `DbContext` is constructed with `contextOwnsConnection: false`, so it borrows the connection without closing it when disposed.
- `childContext.Database.UseTransaction(transaction.UnderlyingTransaction)` enlists the child in the parent's transaction.

```csharp
using (var parentContext = new DatabaseContext(DatabaseContext.ConnectionString)) {
    using (var transaction = parentContext.Database.BeginTransaction(IsolationLevel.ReadCommitted)) {

        // Insert via parent context
        var parent = new Entity() { Name = Guid.NewGuid().ToString() };
        parentContext.Set<Entity>().Add(parent);
        await parentContext.SaveChangesAsync();

        // Child context borrows the same connection (does NOT own it)
        using (var childContext = new DatabaseContext(
                   parentContext.Database.Connection,
                   contextOwnsConnection: false)) {

            // Enlist child in the parent's transaction
            childContext.Database.UseTransaction(transaction.UnderlyingTransaction);

            var reloaded = await childContext.Set<Entity>()
                .SingleOrDefaultAsync(e => e.Id == parent.Id);

            reloaded.Name = $"Name {reloaded.Id}";
            await childContext.SaveChangesAsync();
        }

        transaction.Commit();
        // transaction.Rollback();
    }
}
```

### `DatabaseContext` (EF6)

Exposes two constructors — one accepting a connection string (used by the parent) and one accepting an existing `DbConnection` (used by the child):

```csharp
public DatabaseContext(string connectionString) : base(connectionString) { }

public DatabaseContext(DbConnection dbConnection, bool contextOwnsConnection)
    : base(dbConnection, contextOwnsConnection) { }
```

---

## Entity Framework Core (`EntityFrameworkCore.SharedTransaction`)

### How it works

- The child `DbContextOptions` are built from the **same `DbConnection`** retrieved via `parentContext.Database.GetDbConnection()`.
- `childContext.Database.UseTransactionAsync(transaction.GetDbTransaction())` enlists the child in the parent's transaction.

```csharp
var parentOptions = new DbContextOptionsBuilder()
    .UseSqlServer(DatabaseContext.ConnectionString)
    .Options;

using (var parentContext = new DatabaseContext(parentOptions)) {
    using (var transaction = parentContext.Database.BeginTransaction(IsolationLevel.ReadCommitted)) {

        // Insert via parent context
        var initial = new Entity() { Name = Guid.NewGuid().ToString() };
        await parentContext.Set<Entity>().AddAsync(initial);
        await parentContext.SaveChangesAsync();

        // Build child options reusing the parent's open connection
        var childOptions = new DbContextOptionsBuilder()
            .UseSqlServer(parentContext.Database.GetDbConnection())
            .Options;

        using (var childContext = new DatabaseContext(childOptions)) {
            // Enlist child in the parent's transaction
            await childContext.Database.UseTransactionAsync(transaction.GetDbTransaction());

            var reloaded = await childContext.Set<Entity>()
                .SingleOrDefaultAsync(x => x.Id == initial.Id);

            reloaded.Name = $"Name {reloaded.Id}";
            await childContext.SaveChangesAsync();
        }

        await transaction.CommitAsync();
        // await transaction.RollbackAsync();
    }
}
```

### `DatabaseContext` (EF Core)

Accepts `DbContextOptions`, which allows the caller to supply either a connection string or an existing `DbConnection`:

```csharp
public DatabaseContext(DbContextOptions options) : base(options) { }
```

---

## Project Structure

```
.
└── src/
    ├── EntityFramework.SharedTransaction/          # EF6 console app (.NET Framework)
    │   ├── Database/
    │   │   ├── Schema/
    │   │   │   └── Entity.cs                       # Simple entity with Id and Name
    │   │   └── DatabaseContext.cs                  # DbContext with two constructors (string / DbConnection)
    │   └── Program.cs                              # Demonstrates parent→child transaction sharing (EF6)
    │
    └── EntityFrameworkCore.SharedTransaction/      # EF Core console app (.NET)
        ├── Database/
        │   ├── Schema/
        │   │   └── Entity.cs                       # Simple entity with Id and Name
        │   ├── DatabaseContext.cs                  # DbContext accepting DbContextOptions
        │   └── DatabaseContextFactory.cs           # IDesignTimeDbContextFactory for migrations
        ├── Migrations/                             # EF Core migration files
        └── Program.cs                              # Demonstrates parent→child transaction sharing (EF Core)
```

---

## Key Concepts

| Concept | EF6 | EF Core |
|---|---|---|
| Share connection | Pass `DbConnection` to child constructor with `contextOwnsConnection: false` | Build child `DbContextOptions` from `GetDbConnection()` |
| Enlist in transaction | `UseTransaction(transaction.UnderlyingTransaction)` | `UseTransactionAsync(transaction.GetDbTransaction())` |
| Commit | `transaction.Commit()` | `await transaction.CommitAsync()` |
| Rollback | `transaction.Rollback()` | `await transaction.RollbackAsync()` |

---

## Additional Resources

- [Connection management – EF6](https://learn.microsoft.com/en-us/ef/ef6/fundamentals/connection-management)
- [Transactions in EF Core](https://learn.microsoft.com/en-us/ef/core/saving/transactions)

---

## Prerequisites

- SQL Server LocalDB (ships with Visual Studio): `(localdb)\MSSQLLocalDB`
  - EF6 project creates the database automatically on first run (`CreateIfNotExists`)
  - EF Core project applies migrations automatically on first run (`MigrateAsync`)
- [.NET Framework](https://dotnet.microsoft.com/download/dotnet-framework) for the EF6 project
- [.NET SDK](https://dotnet.microsoft.com/download) for the EF Core project

## Running the Projects

**EF6:**
```bash
cd src/EntityFramework.SharedTransaction
# Open in Visual Studio or build with MSBuild
```

**EF Core:**
```bash
cd src/EntityFrameworkCore.SharedTransaction
dotnet run
```