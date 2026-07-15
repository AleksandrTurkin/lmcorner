+++
date = '2026-07-11T01:04:10+02:00'
title = 'Agentic Coding. Part 2.2: Cursor Pagination'
tags = ["best-practices", "software-design", "dotnet", "jardi-tips"]
author = ["Aleksandr T."]
+++

### Hello there! 🖖

### Overview

* **Ecosystem:** .NET 10
* **Technologies:** ASP.NET Core Web API, PostgreSQL, Entity Framework Core
* **Improvement:** `Cursor Pagination`

Example of cursor pagination with tuple comparison in `PostgreSQL`:

```postgresql
select id, random_value, created_at
from sample_data
where (created_at, id) < ('2026-07-10 02:17:36.269', '2f9475be-67a9-4afe-8c7e-a0bba409dbea')
order by created_at desc, id desc
limit 100;
```

Implementation of cursor pagination with tuple comparison using `Entity Framework Core`:

```csharp
public async Task<IEnumerable<T>> GetPagerResultAsync(IQueryable<T> query, DateTime? dateTime, Guid? lastId, int limit, CancellationToken cancellationToken)
{
    query = query.OrderByDescending(x => x.CreatedAt)
                    .ThenByDescending(x => x.Id);
    
    if (dateTime == null || lastId == null)
        return await query.Take(limit).AsNoTracking().ToListAsync(cancellationToken);

    query = query.Where(x => EF.Functions.LessThan(
        ValueTuple.Create(x.CreatedAt, x.Id),
        ValueTuple.Create(dateTime, lastId)))
        .Take(limit);

    return await query.AsNoTracking().ToListAsync(cancellationToken);
}
```

[AI skills update](https://github.com/AleksandrTurkin/api-jardi-tips/commit/5482fee44914917e1414c2124420bb9272ae970b) for working with cursor pagination.

## Step-by-Step Implementation

Pagination is one of the most important parts of database optimization. It allows us to efficiently load data from large tables without loading everything at once.

Let's start from a slightly different angle: first, let's see how to build an efficient query at the database level, and then implement this approach in our application using `Entity Framework Core`.

### Page-based pagination

I am sure many of you have seen and implemented standard page-based pagination.

For this, we need:

- Count how many rows there are in the table.
- Decide how many rows should be on one page and calculate how many pages are needed to display all rows.
- Load data for a specific page.

1. Let's count the number of rows in the `sample_data` table. In our case, the table has `1 100 000` rows.

```postgresql
select count(*)
from sample_data
```

This operation is quite lightweight:

```text
1 row(s) fetched - 0.055s
```

2. We define the number of rows per page and calculate how many pages are needed to display all data. In our case, let's say one page contains `100` rows, so the number of pages will be `1 100 000 / 100 = 11 000`.

3. Now let's load data for a specific page, for example, a page almost at the end.

```postgresql
select id, random_value, created_at
from sample_data
order by created_at desc, id desc
limit 100 offset (10900 - 1) * 100;
```

And here it gets interesting: this operation took almost 2 seconds.

```text
100 row(s) fetched - 1.912s
```

Loading 100 rows in 2 seconds is, to put it mildly, slow. But if we load data, for example, from the second page, everything works very fast.

```text
100 row(s) fetched - 0.09s
```

When requesting a deep page, PostgreSQL hits a heavy `Sort` node: the database has to read from disk/memory and sort `1 090 000` rows. It spends almost 2 seconds just to return the final 100 rows at the end.

```text
->  Sort  (cost=155788.90..158538.90 rows=1100000 width=28) (actual time=1681.957..1976.727 rows=1090000.00 loops=1)
```

At the same time, for the second page, early exit optimization works: the parallel `Gather Merge` node processes only 200 rows and finishes in just 0.09 seconds.

```text
->  Gather Merge  (cost=33481.18..161594.19 rows=1099999 width=28) (actual time=192.518..203.305 rows=200.00 loops=1)
```

### Cursor pagination

Cursor pagination allows us to efficiently get the next portion of data without skipping a large number of rows through `OFFSET` every time.

Instead of a page number, we use the values from the last record of the previous page as a cursor for the next page.

To implement cursor pagination, we need:

- the identifier of the last item from the previous page;
- the sorting criteria used to load data.

It is important that sorting is stable. If we sort by `created_at`, then rows with the same `created_at` value may be ordered differently. That is why we use `id` as an additional tie-breaker.

Example implementation of cursor pagination:

```postgresql
select id, random_value, created_at
from sample_data
where created_at < '2026-06-10 09:00:17.135'
   or (created_at = '2026-06-10 09:00:17.135'
       and id < 'fc0852b4-7f1c-47e3-8eda-dd4166cf86be')
order by created_at desc, id desc
limit 100;
```

Query result for a deep page:

```text
100 row(s) fetched - 0.113s
```

Result for the second page:

```text
100 row(s) fetched - 0.154s
```

As we can see, cursor pagination does not have performance degradation when moving to deep pages, unlike standard pagination with `OFFSET`.

PostgreSQL allows us to describe cursor pagination in a more concise way using tuple comparison.

```postgresql
select id, random_value, created_at
from sample_data
where (created_at, id) < ('2026-07-10 02:17:36.269', '2f9475be-67a9-4afe-8c7e-a0bba409dbea')
order by created_at desc, id desc
limit 100;
```

Tuple comparison is not only syntactic sugar, but also a useful optimization.

1. One index seek instead of branching

The `OR` condition makes the PostgreSQL planner evaluate several condition branches. In some cases, this can lead to less optimal plans, for example, `Bitmap Index Scan`. Tuple comparison is seen as one condition, and the database can perform a fast `Index Scan` over a B-Tree index.

2. Better scalability for 3+ columns

If pagination is based on three or more fields, for example `priority`, `created_at`, and `id`, boolean logic with `OR` quickly becomes a bulky condition. A tuple like `(a, b, c) < (x, y, z)` scales much better and remains easy for the database to understand.

3. More stable query plans

Queries with many `OR` conditions can be sensitive to data distribution statistics. If the data distribution changes a lot, the query plan can also change in a bad way. Tuple comparison maps better to the B-Tree index structure and usually gives a more stable plan.

After adding an index, cursor pagination becomes very efficient. It is important that the column order in the tuple matches the column order in the index. Then PostgreSQL can efficiently use the B-Tree index to jump to the required page.

```postgresql
CREATE INDEX ix_sample_data_createdat_id ON sample_data ("created_at" DESC, "id" DESC);
```

As a result, the query executes very fast even on large amounts of data:

```text
100 row(s) fetched - 0.002s
```

### Cursor pagination implementation with tuple comparison

Cursor implementation:

```csharp
public record PagedCursor(DateTime DateTime, Guid LastId)
{
    public static string Encode(DateTime date, Guid lastId)
    {
        var cursor = new PagedCursor(date, lastId);
        var json = JsonSerializer.Serialize(cursor);
        
        return WebEncoders.Base64UrlEncode(Encoding.UTF8.GetBytes(json));
    }

    public static PagedCursor? Decode(string? cursor)
    {
        if (string.IsNullOrWhiteSpace(cursor))
            return null;
        
        try
        {
            var json = Encoding.UTF8.GetString(WebEncoders.Base64UrlDecode(cursor));
            return JsonSerializer.Deserialize<PagedCursor>(json);
        }
        catch
        {
            return null;
        }
    }
}

// The cursor will have a text representation
// "pageContext": eyJEYXRlVGltZSI6IjIwMjYtMDctMTRUMDA6NTI6MzMuNjg5MjI0WiIsIkxhc3RJZCI6IjAxOWY1ZTFjLTdmN2QtNzk1Zi04ZTdkLWVlODU4NmMzZmU2ZSJ9"
```

Implementation of cursor pagination with tuple comparison for `PostgreSQL` using `Entity Framework Core`:

```csharp
public async Task<IEnumerable<T>> GetPagerResultAsync(IQueryable<T> query, DateTime? dateTime, Guid? lastId, int limit, CancellationToken cancellationToken)
{
    query = query.OrderByDescending(x => x.CreatedAt)
                    .ThenByDescending(x => x.Id);
    
    if (dateTime == null || lastId == null)
        return await query.Take(limit).AsNoTracking().ToListAsync(cancellationToken);

    query = query.Where(x => EF.Functions.LessThan(
        ValueTuple.Create(x.CreatedAt, x.Id),
        ValueTuple.Create(dateTime, lastId)))
        .Take(limit);

    return await query.AsNoTracking().ToListAsync(cancellationToken);
}
```

Putting the cursor and pagination together:

```csharp
protected async Task<PagedResult<TDto>> BaseHandle<TDto>(TQuery request, Func<TEntity, TDto> map, CancellationToken cancellationToken)
{
    var repository = unitOfWork.Repository<TEntity>();

    var query = repository.GetAll();
    var pageContext = ParsePageContext(request.PageContext);

    var limit = request.Limit ?? DefaultLimit;
    var items = (await repository.GetPagerResultAsync(query, pageContext.DateTime, pageContext.LastId, limit + 1, cancellationToken)).ToList();

    var hasMore = items.Count > limit;
    if (hasMore)
        items.RemoveAt(items.Count - 1);

    var cursor = hasMore
        ? PagedCursor.Encode(items[^1].CreatedAt, items[^1].Id)
        : null;
    
    return new PagedResult<TDto>(cursor, items.Select(map).ToList());
}

private (DateTime? DateTime, Guid? LastId) ParsePageContext(string? pageContext)
{
    if (string.IsNullOrWhiteSpace(pageContext))
        return (null, null);

    var context = PagedCursor.Decode(pageContext);
    
    if (context == null)
        return (null, null);

    return (context.DateTime, context.LastId);
}

/// items[^1] - the last item in the list, used to create the cursor for the next page.
/// limit + 1 - we request one more item than needed to detect if there is a next page.
/// If there is no extra item, it means we reached the end of the list and do not create the next cursor.
```

### AI skills

As usual, we update our AI skills. This time - so that new CRUD operations are generated with cursor pagination support.

- [command-query-handler-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/command-query-handler-creator/SKILL.md) - skill for creating command and query handlers
- [api-endpoint-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/api-endpoint-creator/SKILL.md) - skill for creating API endpoints
- [db-entity-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/db-entity-creator/SKILL.md) - skill for creating database entities and migration steps

You can find the AI skills changes in [this commit](https://github.com/AleksandrTurkin/api-jardi-tips/commit/5482fee44914917e1414c2124420bb9272ae970b).

### Useful Resources

* [Understanding Cursor Pagination and Why It's So Fast (Deep Dive)](https://milanjovanovic.tech/blog/understanding-cursor-pagination-and-why-its-so-fast-deep-dive)
* [Cursor Pagination is the FASTEST - But you can't use it if...](https://www.youtube.com/watch?v=wepqVpRjp64)
* [THIS ONE Trick Made My Database Query 400x FASTER! (Cursor Pagination EXPOSED)](https://www.youtube.com/watch?v=U3LcKY19z_4)

## Blabber

I understand that the example where the table has more than one million rows and we need to display them by 100 rows across 11 000 pages looks like a physics problem from real life: "Let's imagine we have a weightless seesaw in a vacuum with two children on it" 😁.

This is mostly for demonstration: how performance degradation happens when working with large amounts of data, and why standard page-based pagination can be inefficient in such scenarios.

Cursor pagination has its own limitations. For example, we cannot jump to an arbitrary page. We can only implement data loading as an "infinite scroll": load new records as the user scrolls.

For our future application, this scenario fits perfectly.

#### Thanks! Keep calm and code on! 🚀