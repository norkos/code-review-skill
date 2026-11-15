# C# / .NET Code Review Guide

> A C# / .NET 8 code review guide covering C# 12 features, async programming, EF Core performance, ASP.NET Core best practices, dependency injection, LINQ, and other core topics.

## Table of Contents

- [C# 12 New Features](#c-12-new-features)
- [Async Programming](#async-programming)
- [EF Core Performance](#ef-core-performance)
- [ASP.NET Core Best Practices](#aspnet-core-best-practices)
- [Dependency Injection](#dependency-injection)
- [LINQ Best Practices](#linq-best-practices)
- [Review Checklist](#review-checklist)

---

## C# 12 New Features

### Primary Constructors (Non-record Types)

```csharp
// ❌ Traditional constructor with too much boilerplate
public class ProductService
{
    private readonly ProductDbContext _db;
    private readonly ILogger<ProductService> _logger;

    public ProductService(ProductDbContext db, ILogger<ProductService> logger)
    {
        _db = db;
        _logger = logger;
    }
}

// ✅ Primary constructor - concise dependency injection
public class ProductService(ProductDbContext db, ILogger<ProductService> logger)
{
    public async Task<Product?> GetAsync(int id)
        => await db.Products.FindAsync(id);
}

// ⚠️ Note: primary constructor parameters are not properties and should not be reassigned
// ⚠️ If long-term storage is needed, declare a field explicitly
public class OrderService(OrderDbContext db)
{
    private readonly OrderDbContext _db = db; // explicit capture
}
```

### Collection Expressions

```csharp
// ❌ Traditional collection initialization
int[] nums = new int[] { 1, 2, 3 };
List<string> names = new List<string> { "alice", "bob" };

// ✅ Collection expressions
int[] nums = [1, 2, 3];
List<string> names = ["alice", "bob"];
Span<char> span = ['a', 'b'];

// ✅ Spread operator
int[] merged = [..nums, 4, 5];
```

### Default Lambda Parameters

```csharp
// ❌ Overloaded lambdas
var add = (int a, int b) => a + b;
var addDefault = (int a) => a + 1;

// ✅ Default parameters
var add = (int a, int b = 1) => a + b;
```

---

## Async Programming

> 📖 For general concurrency patterns and cross-language examples, see the [Cross-Language Async and Concurrency Guide](cross-cutting/async-concurrency-patterns.md)

### Task.Wait() / .Result / async void Are Serious Anti-patterns

```csharp
// ❌ Task.Wait() - deadlock risk (synchronously blocking on async work)
public ActionResult<Data> Get(int id)
{
    var data = _service.GetDataAsync(id).Result; // deadlock!
    return Ok(data);
}

// ❌ async void - exceptions cannot be caught and will crash the process
public async void HandleEvent()
{
    await _service.ProcessAsync(); // an exception crashes the process directly
}

// ✅ async Task - async all the way
public async Task<ActionResult<Data>> Get(int id)
{
    var data = await _service.GetDataAsync(id);
    return Ok(data);
}
```

### ConfigureAwait(false) for Library Code

```csharp
// ❌ Library code needlessly captures the SynchronizationContext
public class LibraryService
{
    public async Task<string> GetDataAsync()
    {
        var response = await _httpClient.GetAsync("/api/data");
        return await response.Content.ReadAsStringAsync();
    }
}

// ✅ Library code uses ConfigureAwait(false) to avoid deadlocks
public class LibraryService
{
    public async Task<string> GetDataAsync()
    {
        var response = await _httpClient.GetAsync("/api/data").ConfigureAwait(false);
        return await response.Content.ReadAsStringAsync().ConfigureAwait(false);
    }
}
```

### CancellationToken Propagation

```csharp
// ❌ CancellationToken is dropped
public async Task<List<User>> SearchAsync(string query)
{
    return await _db.Users.Where(u => u.Name.Contains(query)).ToListAsync();
}

// ✅ Pass the CancellationToken through the whole call chain
public async Task<List<User>> SearchAsync(string query, CancellationToken ct = default)
{
    return await _db.Users
        .Where(u => u.Name.Contains(query))
        .ToListAsync(ct);
}
```

### Async Disposal

```csharp
// ❌ Synchronously disposing async resources
public class DataClient : IDisposable
{
    public void Dispose()
    {
        _httpClient.Dispose(); // may drop in-flight requests
    }
}

// ✅ IAsyncDisposable
public class DataClient : IAsyncDisposable
{
    public async ValueTask DisposeAsync()
    {
        await _stream.DisposeAsync();
    }
}

// ✅ Callers use await using
await using var client = new DataClient();
```

---

## EF Core Performance

### N+1 Query Problem

> 📖 For general principles and cross-language solutions, see the [Cross-Language N+1 Query Guide](cross-cutting/n-plus-one-queries.md)

```csharp
// ❌ Classic N+1 - each Blog triggers a query to fetch its Posts
foreach (var blog in await context.Blogs.ToListAsync())
{
    foreach (var post in blog.Posts) // hits the database on every iteration!
    {
        Console.WriteLine(post.Title);
    }
}

// ✅ Eager loading + projection
await foreach (var blog in context.Blogs
    .Select(b => new { b.Url, b.Posts })
    .AsAsyncEnumerable())
{
    foreach (var post in blog.Posts)
        Console.WriteLine(post.Title);
}
```

### Over-fetching (No Projection)

```csharp
// ❌ Loads every column - all fields loaded when only Url is needed
var urls = await context.Blogs.ToListAsync();

// ✅ Project only the fields you need
var urls = await context.Blogs
    .Select(b => b.Url)
    .ToListAsync();
```

### Missing Pagination

```csharp
// ❌ Unbounded result set
var posts = await context.Posts
    .Where(p => p.Title.StartsWith("A"))
    .ToListAsync(); // could be millions of rows!

// ✅ Limit the number of results
var posts = await context.Posts
    .Where(p => p.Title.StartsWith("A"))
    .OrderBy(p => p.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

### Cartesian Explosion (JOINs)

```csharp
// ❌ Multiple Includes produce large amounts of duplicated data
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Tags)
    .ToListAsync(); // Blog data repeated on every row

// ✅ Use AsSplitQuery to split the query
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Tags)
    .AsSplitQuery()
    .ToListAsync();
```

### Missing AsNoTracking for Read-only Scenarios

```csharp
// ❌ Tracking by default - read-only queries still pay the tracking cost
var products = await context.Products.ToListAsync();

// ✅ AsNoTracking - skips change tracking, faster and uses less memory
var products = await context.Products
    .AsNoTracking()
    .ToListAsync();
```

### Functions on Columns Prevent Index Usage

```csharp
// ✅ Can use an index - sargable
var posts1 = await context.Posts
    .Where(p => p.Title.StartsWith("A"))
    .ToListAsync();

// ❌ Cannot use an index - full table scan
var posts2 = await context.Posts
    .Where(p => p.Title.EndsWith("A"))
    .ToListAsync();

// ❌ Function wrapped around the column - full table scan
var posts3 = await context.Posts
    .Where(p => p.Title.ToLower() == "foo")
    .ToListAsync();
```

### Sync vs Async Database Access

```csharp
// ❌ Synchronous database calls - block the thread
var products = context.Products.ToList();
context.SaveChanges();

// ✅ Asynchronous database calls
var products = await context.Products.ToListAsync();
await context.SaveChangesAsync();
```

---

## ASP.NET Core Best Practices

### HttpClient Misuse

```csharp
// ❌ Creating a new HttpClient per request - socket exhaustion
using var client = new HttpClient();
var response = await client.GetAsync("https://api.example.com/data");

// ✅ Inject via IHttpClientFactory
public class MyService
{
    private readonly HttpClient _client;
    public MyService(HttpClient client) => _client = client; // injected from the factory
}
```

### Using HttpContext on Background Threads

```csharp
// ❌ Capturing a scoped service in a background task - already disposed once the request ends
_ = Task.Run(async () =>
{
    await context.SaveChangesAsync(); // ObjectDisposedException!
});

// ✅ Create a new scope
_ = Task.Run(async () =>
{
    await using var scope = serviceScopeFactory.CreateAsyncScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.SaveChangesAsync();
});
```

### Synchronous Request.Form Access

```csharp
// ❌ Reading Form synchronously - sync over async
var form = HttpContext.Request.Form;

// ✅ Read asynchronously
var form = await HttpContext.Request.ReadFormAsync();
```

### Exceptions for Control Flow

```csharp
// ❌ Using exceptions to check existence - exceptions are expensive, much slower than a direct check
try
{
    var user = await _db.Users.FirstAsync(u => u.Id == id);
}
catch (InvalidOperationException)
{
    return NotFound();
}

// ✅ Use a check instead of an exception
var user = await _db.Users.FirstOrDefaultAsync(u => u.Id == id);
if (user is null) return NotFound();
```

### Response Headers Set After the Body

```csharp
// ❌ Setting a header after the body has been sent - throws
await next(context);
context.Response.Headers["X-Custom"] = "value"; // may throw!

// ✅ Use the OnStarting callback
context.Response.OnStarting(() =>
{
    context.Response.Headers["X-Custom"] = "value";
    return Task.CompletedTask;
});
await next(context);
```

---

## Dependency Injection

### Injecting Scoped Services into Singletons

```csharp
// ❌ Scoped service injected into a Singleton - lifetime mismatch
services.AddSingleton<BackgroundWorker>();
services.AddScoped<IUserRepository, UserRepository>();

// BackgroundWorker is a Singleton, UserRepository is Scoped
// → UserRepository gets shared across requests or is already disposed

// ✅ Create a scope inside the Singleton via IServiceProvider
public class BackgroundWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public BackgroundWorker(IServiceScopeFactory scopeFactory)
        => _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await using var scope = _scopeFactory.CreateAsyncScope();
        var repo = scope.ServiceProvider.GetRequiredService<IUserRepository>();
    }
}
```

---

## LINQ Best Practices

### LINQ After ToList

```csharp
// ❌ ToList first, then filter - loads the whole table into memory
var results = context.Posts
    .Where(p => p.Title.StartsWith("A"))
    .ToList()
    .Where(p => SomeClientFilter(p)); // client-side filtering, all rows already loaded

// ✅ Let the database do as much filtering as possible
var results = await context.Posts
    .Where(p => p.Title.StartsWith("A") && SomeDbFilter(p))
    .AsAsyncEnumerable()
    .Where(p => SomeClientFilter(p)) // only filters rows returned by the database
    .ToListAsync();
```

### Count() vs Any()

```csharp
// ❌ Count() runs the full query
if (context.Users.Count() > 0) { /* ... */ }

// ✅ Any() is more efficient - returns as soon as the first record is found
if (await context.Users.AnyAsync()) { /* ... */ }
```

### Multiple Enumeration of IEnumerable

```csharp
// ❌ IEnumerable is enumerated twice
public void Process(IEnumerable<int> numbers)
{
    if (numbers.Any()) // first enumeration
    {
        foreach (var n in numbers) // second enumeration (may re-run the query)
        {
            Console.WriteLine(n);
        }
    }
}

// ✅ Materialize first if it will be used more than once
public void Process(IEnumerable<int> numbers)
{
    var list = numbers.ToList(); // enumerated only once
    if (list.Any())
    {
        foreach (var n in list)
        {
            Console.WriteLine(n);
        }
    }
}
```

### Side Effects in Select

```csharp
// ❌ Side effects in Select - unpredictable execution timing
var results = users.Select(u =>
{
    _logger.LogInformation($"Processing {u.Name}"); // side effect!
    return u.Email;
}).ToList();

// ✅ Put side effects in a foreach
foreach (var user in users)
{
    _logger.LogInformation("Processing {Name}", user.Name);
}
var results = users.Select(u => u.Email).ToList();
```

---

## Review Checklist

### C# 12 New Features

- [ ] Primary constructor parameters are not reassigned
- [ ] Collection expression syntax is consistent (old and new styles are not mixed)

### Async Programming

- [ ] No `Task.Wait()`, `.Result`, or `async void`
- [ ] Library code uses `ConfigureAwait(false)`
- [ ] `CancellationToken` is passed through the whole call chain
- [ ] Async resources use `IAsyncDisposable` / `await using`
- [ ] Sync and async data access are not mixed

### EF Core

- [ ] No N+1 queries (navigation properties accessed inside loops)
- [ ] Projection with `Select()` avoids over-fetching
- [ ] Pagination: `Take()`/`Skip()` before `ToListAsync()`
- [ ] Multiple `Include()` calls use `AsSplitQuery()`
- [ ] Read-only queries use `AsNoTracking()`
- [ ] No function calls on columns that prevent index usage
- [ ] All database calls are async

### ASP.NET Core

- [ ] HttpClient is obtained via `IHttpClientFactory`
- [ ] Scoped services are not used directly in background tasks
- [ ] `ReadFormAsync` is used instead of `Request.Form`
- [ ] Exceptions are not used for control flow
- [ ] Response headers are set via `OnStarting`

### Dependency Injection

- [ ] Scoped services are not injected into Singletons
- [ ] Background tasks create a new scope

### LINQ

- [ ] No unnecessary `ToList()` followed by more LINQ
- [ ] `Any()` instead of `Count() > 0`
- [ ] IEnumerable is not enumerated multiple times (or is materialized first)
- [ ] No side effects in Select
