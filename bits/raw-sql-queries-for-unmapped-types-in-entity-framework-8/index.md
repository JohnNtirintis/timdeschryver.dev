---
title: Raw SQL Queries for Unmapped Types in Entity Framework 8
slug: raw-sql-queries-for-unmapped-types-in-entity-framework-8
date: 2023-08-15
tags: dotnet, entity-framework
---

# Raw SQL Queries for Unmapped Types in Entity Framework 8

Entity Framework 8 has a new feature that allows you to execute raw SQL queries against the database and return results as unmapped types.
To use this feature, use the new `SqlQuery` method on the `Database` property of your `DbContext` instance.

This feature is useful when you want your query to return a specific type for a specific purpose.
For example, in many cases you don't need/want the overhead of returning your full-blown entity for search queries.
Instead, you want a optimized entity (e.g. a DTO) that **only** contains the data you need for that specific purpose.
Usually this results in a faster query and less data transferred over the wire.

See my blog post [You can now return unmapped types from raw SQL select statements with Entity Framework 8](../../blog/you-can-now-return-unmapped-types-from-raw-sql-select-statements-with-entity-framework-8/index.md) for more info about this new feature.

:::code-group

```csharp:Select Query to retrieve a collection [title=C# Code]
var customers  = await dbContext.Database
    // 👇 Map to a unmapped type
    .SqlQuery<CustomerDto>(
        $"""
            SELECT
                c.Id as CustomerId,
                c.FirstName,
                a.Street
            FROM dbo.Customers c
            JOIN dbo.Addresses a ON c.Id = a.CustomerId
            """
    )
    .ToListAsync();
```

```sql:Select Query to retrieve a collection [title=Generated SQL]
SELECT
       c.Id as CustomerId,
       c.FirstName,
       a.Street
FROM dbo.Customers c
JOIN dbo.Addresses a ON c.Id = a.CustomerId
```

:::

:::code-group

```csharp:Select Query with a parameter within the where clause [title=C# Code]
var customers = await dbContext.Database
    .SqlQuery<CustomerDto>(
        $"""
            SELECT
                c.Id as CustomerId,
                c.FirstName,
                a.Street
            FROM dbo.Customers c
            JOIN dbo.Addresses a ON c.Id = a.CustomerId
            --                             👇 Use parameters in your query
            WHERE c.FirstName like '%' + {customerName} + '%'
            """
    )
    .ToListAsync();
```

```sql:Select Query with a parameter within the where clause [title=Generated SQL]
exec sp_executesql N'SELECT
       c.Id as CustomerId,
       c.FirstName,
       a.Street
FROM dbo.Customers c
JOIN dbo.Addresses a ON c.Id = a.CustomerId
--                             👇 Parameterized
WHERE c.FirstName like ''%'' + @p0 + ''%''
',N'@p0 nvarchar(4000)',@p0=N'ali'
```

:::

:::code-group

```csharp:Select Query using LINQ to retrieve a single entity [title=C# Code]
var customer = await dbContext.Database
    .SqlQuery<CustomerDto>(
        $"""
            SELECT
                c.Id as CustomerId,
                c.FirstName,
                a.Street
            FROM dbo.Customers c
            JOIN dbo.Addresses a ON c.Id = a.CustomerId
            """
    )
    // 👇 SqlQuery returns a IQueryable<TResult> so you can use LINQ as well
    .SingleOrDefaultAsync(c => c.CustomerId == customerId);
```

```sql:Select Query using LINQ to retrieve a single entity [title=Generated SQL]
exec sp_executesql N'SELECT TOP(2) [c].[CustomerId], [c].[FirstName], [c].[Street]
FROM (
    SELECT
        c.Id as CustomerId,
        c.FirstName,
        a.Street
    FROM dbo.Customers c
    JOIN dbo.Addresses a ON c.Id = a.CustomerId
) AS [c]
--                       👇 LINQ filters also are parameterized
WHERE [c].[CustomerId] = @__customerId_1',N'@__customerId_1 int',@__customerId_1=1
```

:::
