
# 📚 Entity Framework Core - The Complete Developer's Guide

A comprehensive, structured guide covering the fundamental concepts, advanced configurations, and best practices for working with Entity Framework Core (EF Core). Use this as a reference for learning and revision.

---

## Table of Contents

### I. Setup & Basics
1.  [DbContext and Registration](#1-dbcontext-and-registration)
2.  [Mapping Classes to Tables](#2-mapping-classes-to-tables)
3.  [Creating and Managing Migrations](#3-creating-and-managing-migrations)
4.  [Data Seeding](#4-data-seeding)

### II. Modeling & Configuration
5.  [Configuration Methods: Fluent API vs. Data Annotations](#5-configuration-methods-fluent-api-vs-data-annotations)
6.  [Primary Key Configuration](#6-primary-key-configuration)
7.  [Relationships and Navigation Properties](#7-relationships-and-navigation-properties)
8.  [Database Schema & Structure Customization](#8-database-schema--structure-customization)
9.  [Property and Column Configuration](#9-property-and-column-configuration)
10. [Excluding Entities and Properties](#10-excluding-entities-and-properties)
11. [Indexes and Sequences](#11-indexes-and-sequences)

### III. Data Operations (CRUD & LINQ)
12. [Selecting and Filtering Data (LINQ Basics)](#12-selecting-and-filtering-data-linq-basics)
13. [Aggregate Functions and Grouping](#13-aggregate-functions-and-grouping)
14. [Eager, Explicit, and Lazy Loading](#14-eager-explicit-and-lazy-loading)
15. [Tracking and Performance (`AsNoTracking`)](#15-tracking-and-performance-asnotracking)
16. [Inserting, Updating, and Deleting Entities](#16-inserting-updating-and-deleting-entities)
17. [Raw SQL, Stored Procedures, and Transactions](#17-raw-sql-stored-procedures-and-transactions)

### IV. Advanced Features & Best Practices
18. [Computed Columns and Default Values](#18-computed-columns-and-default-values)
19. [Global Query Filters](#19-global-query-filters)
20. [Best Practices and Resources](#20-best-practices-and-resources)

---

## I. Setup & Basics

### 1. DbContext and Registration

The **`DbContext`** is the primary class in EF Core, acting as the session with the database. It handles database connections, transaction management, and mapping domain entities to database tables. **`DbSet<T>`** properties represent the collections of entities that will map to tables in your database.

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    // DbSet properties map to database tables
    public DbSet<Post> Posts { get; set; }
    public DbSet<User> Users { get; set; }
    public DbSet<Comment> Comments { get; set; }

    // Use OnConfiguring to set the connection string if not using DI
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            optionsBuilder.UseSqlServer("YourConnectionString");
        }
    }

    // OnModelCreating is where Fluent API configurations are applied
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        // Fluent API configurations go here
    }
}
````

**Registering in Program.cs / Startup.cs (Recommended using Dependency Injection):**

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

-----

### 2\. Mapping Classes to Tables

By **convention**, EF Core automatically maps public classes to database tables and properties to columns. The table name is typically the pluralized name of the class (e.g., `Post` class maps to the `Posts` table).

```csharp
public class Post
{
    public int Id { get; set; } // PK by convention
    public string Title { get; set; }
    public string Content { get; set; }
    public DateTime CreatedDate { get; set; }
}
```

-----

### 3\. Creating and Managing Migrations

**Migrations** allow you to evolve your database schema as your model changes. They are version control for your database.

#### Creating Migrations

This command scaffolds a new migration file based on changes to your `DbContext`.

```bash
# Using .NET CLI
dotnet ef migrations add InitialCreate
```

#### Applying Migrations

This command applies the pending migrations to your database.

```bash
# .NET CLI
dotnet ef database update
```

#### Removing Migrations

This command reverts the last migration *that has not been applied* to the database.

```bash
# .NET CLI
dotnet ef migrations remove
```

#### Generating SQL From Migrations

You can view the raw SQL that a migration will execute, which is useful for deployment.

```bash
# .NET CLI
dotnet ef migrations script
```

-----

### 4\. Data Seeding

**Data Seeding** is the process of populating a database with an initial set of data. This is typically used for reference data (e.g., Categories, Countries).

#### Using `HasData` in `OnModelCreating` (Recommended)

This approach integrates the seed data directly into your migrations.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Seed data
    modelBuilder.Entity<Post>().HasData(
        new Post { Id = 1, Title = "First Post", Content = "Content 1", CreatedDate = DateTime.Now },
        new Post { Id = 2, Title = "Second Post", Content = "Content 2", CreatedDate = DateTime.Now }
    );
}
```

#### Using Raw SQL in an Empty Migration

For complex seeding or data that requires database functions (like `GETDATE()`):

```csharp
public partial class SeedData : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql(@"
            INSERT INTO Posts (Title, Content, CreatedDate)
            VALUES ('First Post', 'This is the first post', GETDATE())
        ");
    }
    // ... Down method for reversing
}
```

-----

## II. Modeling & Configuration

### 5\. Configuration Methods: Fluent API vs. Data Annotations

EF Core provides two main ways to configure your model: **Data Annotations** (attributes on your entity classes) and the **Fluent API** (method calls inside `DbContext.OnModelCreating`).

| Feature | Data Annotations | Fluent API |
| :--- | :--- | :--- |
| **Simplicity** | Simpler, less code, easy to read | More verbose |
| **Power** | Limited functionality, less flexible | Supports **all** configurations (e.g., composite keys, many-to-many joins) |
| **Separation** | Mixes configuration with the domain model | Better **separation of concerns** |
| **Location** | Applied directly to properties/classes | Defined in the `DbContext` |

#### Example: Setting a Property as Required

**Data Annotations:**

```csharp
public class Post
{
    [Required]
    public string Title { get; set; }
}
```

**Fluent API:**

```csharp
modelBuilder.Entity<Post>(entity =>
{
    entity.Property(p => p.Title)
        .IsRequired()
        .HasMaxLength(200);
});
```

-----

### 6\. Primary Key Configuration

A **Primary Key (PK)** is a column or set of columns that uniquely identifies a row in a table.

#### Convention-Based PKs

EF Core automatically uses `Id` or `{ClassName}Id` as the primary key.

#### Using Data Annotations

Use `[Key]` when the primary key property name does not follow convention.

```csharp
public class Post
{
    [Key]
    public int PostIdentifier { get; set; } 
}
```

#### Using Fluent API (Single Key)

```csharp
modelBuilder.Entity<Post>()
    .HasKey(p => p.PostIdentifier)
    .Property(x => x.PostIdentifier)
    .ValueGeneratedOnAdd(); // for auto-increment identity
```

#### Composite Primary Key (Fluent API Only)

A key composed of multiple columns.

```csharp
public class OrderItem
{
    public int OrderId { get; set; }
    public int ProductId { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<OrderItem>()
        .HasKey(oi => new { oi.OrderId, oi.ProductId });
}
```

-----

### 7\. Relationships and Navigation Properties

Relationships define how entities relate to each other (One-to-One, One-to-Many, Many-to-Many). **Navigation properties** allow you to navigate these relationships in code.

#### One-to-Many Configuration

```csharp
public class Post
{
    public int Id { get; set; }
    // Foreign Key
    public int AuthorId { get; set; } 
    // Navigation property (One)
    public User Author { get; set; }
    // Collection navigation property (Many)
    public ICollection<Comment> Comments { get; set; } 
}

// Fluent API
modelBuilder.Entity<Post>()
    .HasOne(p => p.Author) // Post has One Author
    .WithMany(u => u.Posts) // Author has Many Posts
    .HasForeignKey(p => p.AuthorId);
```

#### One-to-One Configuration (Fluent API)

```csharp
// Assumes UserProfile's PK is also its FK to User
builder.Entity<User>()
    .HasOne(u => u.Profile)
    .WithOne(p => p.User)
    .HasForeignKey<UserProfile>(p => p.UserId);
```

#### Many-to-Many (Fluent API with Join Entity)

When you have a join entity (e.g., `StudentCourse`) to hold additional data.

```csharp
modelBuilder.Entity<StudentCourse>()
    .HasKey(sc => new { sc.StudentId, sc.CourseId });
```

-----

### 8\. Database Schema & Structure Customization

You can rename tables, change the database schema, and even map an entity to a database view.

#### Renaming Tables

**Data Annotations:**

```csharp
[Table("posts")]
public class Post { }
```

**Fluent API:**

```csharp
modelBuilder.Entity<Post>()
    .ToTable("posts");
```

#### Setting Custom Schema for a Table

**Data Annotations:**

```csharp
[Table("posts", Schema = "blog")]
public class Post { }
```

**Fluent API:**

```csharp
modelBuilder.Entity<Post>()
    .ToTable("posts", schema: "blog");
```

#### Setting Default Schema

```csharp
modelBuilder.HasDefaultSchema("blog");
```

#### Mapping to a View

```csharp
modelBuilder.Entity<PostSummary>()
    .ToView("vw_PostSummary")
    .HasNoKey(); // Views typically don't have keys
```

-----

### 9\. Property and Column Configuration

Configuration methods to control the column characteristics like name, data type, and size.

| Goal | Data Annotations | Fluent API |
| :--- | :--- | :--- |
| **Required** | `[Required]` | `.IsRequired()` |
| **Max Length** | `[MaxLength(200)]` | `.HasMaxLength(200)` |
| **Rename Column** | `[Column("post_title")]` | `.HasColumnName("post_title")` |
| **Change Column Type** | `[Column(TypeName = "decimal(18,2)")]` | `.HasColumnType("decimal(18,2)")` |
| **Add Comment** | `[Comment("Post title")]` | `.HasComment("Post title")` |

-----

### 10\. Excluding Entities and Properties

Use this to prevent an entity or property from being mapped to the database.

#### Excluding an Entire Class

**Data Annotations:**

```csharp
[NotMapped]
public class AuditLog { }
```

**Fluent API:**

```csharp
modelBuilder.Ignore<AuditLog>();
```

#### Excluding a Single Property

**Data Annotations:**

```csharp
public class Post
{
    [NotMapped]
    public string TemporaryData { get; set; }
}
```

**Fluent API:**

```csharp
modelBuilder.Entity<Post>()
    .Ignore(p => p.TemporaryData);
```

-----

### 11\. Indexes and Sequences

#### Indexes

**Indexes** improve the speed of data retrieval operations on a table.

**Data Annotation:**

```csharp
[Index(nameof(Name), IsUnique = true)]
public class Category { }
```

**Fluent API:**

```csharp
builder.HasIndex(x => x.Name)
    .IsUnique()
    .HasDatabaseName("IX_CustomName")
    .HasFilter("[IsDeleted]=0"); // Example of filtered index
```

#### Sequences

**Sequences** generate sequential values outside the context of a specific table, useful for things like shared invoice numbers.

```csharp
modelBuilder.HasSequence<int>("InvoiceNumbers", schema: "shared")
    .StartsAt(1000)
    .IncrementsBy(1);

// Use in property:
.Property(x => x.InvoiceNo)
    .HasDefaultValueSql("NEXT VALUE FOR shared.InvoiceNumbers");
```

-----

## III. Data Operations (CRUD & LINQ)

### 12\. Selecting and Filtering Data (LINQ Basics)

**Language Integrated Query (LINQ)** is the primary method for querying data in EF Core.

| Method | Description | SQL Translation |
| :--- | :--- | :--- |
| `ToList()` | Executes the query and returns all results as a list. | `SELECT ...` |
| `Where(x => ...)` | Filters data based on a predicate. | `WHERE ...` |
| `Find()` | Retrieves an entity by its primary key (PK). | `SELECT TOP 1 ... WHERE PK = @pk` |
| `Single() / SingleOrDefault()` | Returns one entity or throws an exception/returns null if not found. | `SELECT TOP 2 ...` |
| `First() / FirstOrDefault()` | Returns the first entity matching the criteria. | `SELECT TOP 1 ...` |
| `Take() / Skip()` | Used for **pagination** (limit and offset). | `OFFSET ... ROWS FETCH NEXT ... ROWS ONLY` |
| `OrderBy() / ThenBy()` | Sorts the results. | `ORDER BY ...` |
| `Distinct()` | Returns only unique elements in the final projection. | `SELECT DISTINCT ...` |
| `Select()` | Projects the results into a new shape (anonymous type or DTO). | `SELECT ...` |

**Example:**

```csharp
var items = ctx.Products
    .Where(x => x.Price > 100)
    .OrderByDescending(x => x.Price)
    .Select(x => new { x.Name, x.Price })
    .ToList();
```

-----

### 13\. Aggregate Functions and Grouping

These functions execute on the server side (translated to SQL) for efficiency.

| Function | Description |
| :--- | :--- |
| `Average()` | Calculates the average value. |
| `Count() / LongCount()` | Counts the number of elements. |
| `Sum()` | Calculates the sum of values. |
| `Min() / Max()` | Finds the minimum/maximum value. |
| `Any() / All()` | Checks for existence / if all elements satisfy a condition. |

**Grouping Example:**

```csharp
var q = ctx.Products
    .GroupBy(x => x.CategoryId)
    .Select(g => new { CategoryId = g.Key, Count = g.Count() });
```

-----

### 14\. Eager, Explicit, and Lazy Loading

These control how related entities (navigation properties) are loaded from the database.

#### Eager Loading (Recommended)

Loads related data as part of the initial query using `Include()` and `ThenInclude()`.

```csharp
ctx.Orders.Include(x => x.User)
          .ThenInclude(x => x.Address) // Chain loading
          .ToList();
```

#### Explicit Loading

Manually loads related data for an entity that is already tracked by the context.

```csharp
// Load a single reference
context.Entry(order)
    .Reference(x => x.User)
    .Load(); 

// Load a collection with filtering
context.Entry(order)
    .Collection(x => x.Items)
    .Query().Where(i => i.Price > 100)
    .Load();
```

#### Lazy Loading

Related data is loaded automatically the first time a navigation property is accessed. This can cause the **N+1 problem** (many small queries).
*Requires:* **Virtual** navigation properties and `UseLazyLoadingProxies()`.

-----

### 15\. Tracking and Performance (`AsNoTracking`)

#### Entity Tracking

By default, EF Core tracks entities loaded from the database. This allows it to detect changes and save them when `SaveChanges()` is called. The entity state can be viewed using `context.Entry(entity).State`.

#### Disabling Tracking for Read-Only Queries

Use `AsNoTracking()` for queries where you do not intend to update the data. This significantly improves performance because the change tracker is bypassed.

```csharp
ctx.Products.AsNoTracking();
```

#### Split Queries

Used to avoid generating very large and complex `JOIN` statements when eager loading many collections, by executing multiple simpler queries instead.

```csharp
ctx.Orders.Include(x => x.Items)
          .AsSplitQuery();
```

-----

### 16\. Inserting, Updating, and Deleting Entities

#### Inserting

```csharp
ctx.Products.Add(new Product { Name = "New Product" });
ctx.Products.AddRange(newProductList);
await ctx.SaveChangesAsync();
```

#### Updating

`Update()` marks all properties as modified, which is inefficient. For optimal updates, load the entity, modify only the necessary properties, and call `SaveChanges()`.

```csharp
// Bad: Marks all properties as modified, potentially overwriting nulls/defaults
ctx.Products.Update(entityFromDTO); 

// Good: Only mark specific properties as modified
context.Entry(entity).Property(p => p.Name).IsModified = true; 
await ctx.SaveChangesAsync();
```

#### Deleting

**Cascade Delete** is the default behavior for relationships, meaning related entities are deleted automatically.

```csharp
ctx.Products.Remove(productToDelete);
ctx.Products.RemoveRange(listToRemove);
await ctx.SaveChangesAsync();

// Relationship Configuration
.OnDelete(DeleteBehavior.Cascade) // Related entities are deleted
.OnDelete(DeleteBehavior.Restrict) // Deletion is blocked if related entities exist
.OnDelete(DeleteBehavior.SetNull) // Foreign key set to NULL
```

-----

### 17\. Raw SQL, Stored Procedures, and Transactions

#### Raw SQL

Executes raw SQL queries that return entities tracked by the `DbContext`.

```csharp
// Returns Product entities
ctx.Products.FromSqlRaw("SELECT * FROM Products WHERE Price > 100").ToList(); 
```

#### Stored Procedures

Can be called using `FromSqlRaw` or `FromSqlInterpolated`.

```csharp
var param = new SqlParameter("count", 10);
ctx.Products.FromSqlRaw("EXEC GetTopProducts @count", param).ToList();
```

#### Transactions

Guarantees that a series of operations are treated as a single unit—either all succeed or all fail.

```csharp
using var transaction = await ctx.Database.BeginTransactionAsync();
try
{
    // database operations (Add, Update, Remove)
    await ctx.SaveChangesAsync(); 
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
}
```

-----

## IV. Advanced Features & Best Practices

### 18\. Computed Columns and Default Values

#### Default Values

Used to set a value for a column when a new row is inserted, if a value wasn't provided in the entity.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Post>(entity =>
    {
        // Default value set by a SQL expression (e.g., database time)
        entity.Property(p => p.CreatedDate)
            .HasDefaultValueSql("GETDATE()"); 

        // Default value set by a constant value
        entity.Property(p => p.ViewCount)
            .HasDefaultValue(0); 
    });
}
```

#### Computed Columns

Columns whose value is calculated by the database based on an expression, usually involving other columns.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Product>(entity =>
    {
        // FullName is calculated on-the-fly when read (virtual)
        entity.Property(p => p.FullName)
            .HasComputedColumnSql("[FirstName] + ' ' + [LastName]");

        // FullName is calculated and physically stored in the database (stored)
        entity.Property(p => p.FullName)
            .HasComputedColumnSql("[FirstName] + ' ' + [LastName]", stored: true);
    });
}
```

-----

### 19\. Global Query Filters

**Global Query Filters** apply `WHERE` clauses automatically to all LINQ queries for specific entities, making it easy to implement soft-deletion or multi-tenancy.

```csharp
// Example: Automatically filter out 'deleted' entities
modelBuilder.Entity<Product>()
    .HasQueryFilter(x => !x.IsDeleted); 
```

> **Note:** Filters can be disabled for a query using `IgnoreQueryFilters()`.

-----

### 20\. Best Practices and Resources

1.  **Use Fluent API for complex configurations** for better separation of concerns and power.
2.  **Use Data Annotations for simple validations** (`[Required]`, `[MaxLength]`) for quick readability.
3.  **Always use migrations** to manage your database schema evolution.
4.  **Use `AsNoTracking()`** for all read-only queries.
5.  **Use Eager Loading (`Include`)** to load related data to avoid the N+1 problem.
6.  **Seed essential reference data only** through `HasData()`.
7.  **Organize configurations** using `IEntityTypeConfiguration<T>` in large projects.

#### Additional Resources

  - [Official EF Core Documentation](https://docs.microsoft.com/en-us/ef/core/)
  - [EF Core GitHub Repository](https://github.com/dotnet/efcore)
