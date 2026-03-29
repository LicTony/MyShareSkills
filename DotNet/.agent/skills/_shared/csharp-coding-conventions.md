# C# Coding Conventions (.NET 10)

> **Version:** 1.0.0 — 2026-03-24
> These conventions apply to all C# code targeting .NET 10. They are not suggestions — they are the standard. Deviations require explicit justification.

---

## 1. Naming Conventions

### General Rules

| Element | Convention | Example |
|---|---|---|
| Class | `PascalCase` | `OrderProcessor` |
| Interface | `IPascalCase` | `IOrderRepository` |
| Record | `PascalCase` | `CustomerDto` |
| Struct | `PascalCase` | `Coordinate` |
| Enum | `PascalCase` | `OrderStatus` |
| Enum member | `PascalCase` | `OrderStatus.Pending` |
| Method | `PascalCase` | `CalculateTotal()` |
| Property | `PascalCase` | `FirstName` |
| Event | `PascalCase` | `OrderPlaced` |
| Constant (`const`) | `PascalCase` | `MaxRetryCount` |
| Static readonly field | `PascalCase` | `DefaultTimeout` |
| Private field | `_camelCase` | `_orderRepository` |
| Local variable | `camelCase` | `itemCount` |
| Parameter | `camelCase` | `customerId` |
| Generic type param | `T` or `TPascalCase` | `TEntity`, `TResult` |
| Namespace | `PascalCase` | `MyApp.Domain.Orders` |

### Specific Rules

- **No Hungarian notation.** `strName`, `intCount`, `bIsValid` → banned.
- **No abbreviations** unless universally understood (`Id`, `Url`, `Http`, `Dto`, `Vm`).
- **Async methods** MUST end in `Async`: `GetOrderAsync()`, not `GetOrder()`.
- **Boolean properties/vars** use `is`, `has`, `can`, `should` prefix: `isActive`, `hasPermission`.
- **Avoid noise words**: `Manager`, `Helper`, `Util`, `Handler` are red flags — they mean you haven't thought hard enough about the abstraction.

---

## 2. File & Type Organization

### One type per file
Each file contains exactly ONE top-level type (class, interface, record, enum). No exceptions.

### File name matches type name
`OrderProcessor.cs` → contains `class OrderProcessor`. Period.

### Namespace matches folder structure
```
MyApp/
  Domain/
    Orders/
      Order.cs          → namespace MyApp.Domain.Orders
      OrderStatus.cs    → namespace MyApp.Domain.Orders
  Application/
    Orders/
      PlaceOrderCommand.cs → namespace MyApp.Application.Orders
```

### Member ordering within a type
1. Constants and static readonly fields
2. Private fields
3. Constructors
4. Public properties
5. Public methods
6. Protected methods
7. Private methods
8. Nested types (avoid unless justified)

---

## 3. Language Features (.NET 10 / C# 13+)

### Use modern syntax — no excuses

```csharp
// ✅ File-scoped namespaces (ALWAYS)
namespace MyApp.Domain.Orders;

// ✅ Primary constructors for simple DI
public class OrderService(IOrderRepository repository, ILogger<OrderService> logger)
{
    public async Task<Order> GetAsync(Guid id, CancellationToken ct = default)
    {
        var order = await repository.FindByIdAsync(id, ct)
            ?? throw new NotFoundException($"Order {id} not found.");
        return order;
    }
}

// ❌ Old-style constructor boilerplate for simple DI
public class OrderService
{
    private readonly IOrderRepository _repository;
    public OrderService(IOrderRepository repository) => _repository = repository;
}
```

### Records for immutable data

```csharp
// ✅ Use records for DTOs, value objects, results
public record OrderDto(Guid Id, string CustomerName, decimal Total);

public record CreateOrderCommand(Guid CustomerId, IReadOnlyList<OrderLineDto> Lines);

// ✅ Value objects as records with validation
public record Money(decimal Amount, string Currency)
{
    public static Money Create(decimal amount, string currency)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(amount);
        ArgumentException.ThrowIfNullOrWhiteSpace(currency);
        return new(amount, currency);
    }
}
```

### Pattern matching — use it properly

```csharp
// ✅ Switch expressions for transformation
string Describe(OrderStatus status) => status switch
{
    OrderStatus.Pending   => "Awaiting confirmation",
    OrderStatus.Confirmed => "Being processed",
    OrderStatus.Shipped   => "On the way",
    OrderStatus.Delivered => "Completed",
    _                     => throw new ArgumentOutOfRangeException(nameof(status))
};

// ✅ Property patterns
bool IsEligibleForDiscount(Order order) =>
    order is { Status: OrderStatus.Confirmed, Total.Amount: > 100m };
```

### Null handling

```csharp
// ✅ Nullable reference types ENABLED (always, in every project)
// In .csproj: <Nullable>enable</Nullable>

// ✅ Null-coalescing for defaults
var name = customer.Name ?? "Unknown";

// ✅ Null-conditional for safe navigation
var city = customer?.Address?.City;

// ✅ ArgumentNullException.ThrowIfNull (not manual null checks)
ArgumentNullException.ThrowIfNull(order);

// ❌ Never suppress nullable warnings without a comment explaining why
var result = GetResult()!; // Only if you KNOW it cannot be null and explain why
```

### Collections

```csharp
// ✅ Use collection expressions (C# 12+)
int[] numbers = [1, 2, 3, 4, 5];
List<string> names = ["Alice", "Bob"];

// ✅ Prefer IReadOnlyList<T> / IReadOnlyCollection<T> for parameters and returns
public IReadOnlyList<Order> GetOrders() { ... }

// ✅ Use IEnumerable<T> only for deferred/lazy sequences
// Use IReadOnlyList<T> when you need count or indexing

// ❌ Never expose List<T>, Dictionary<K,V> directly in public APIs
public List<Order> Orders { get; set; } // ❌ mutable collection leak
```

---

## 4. Async/Await

### The non-negotiables

```csharp
// ✅ ALWAYS propagate CancellationToken
public async Task<Customer> GetCustomerAsync(Guid id, CancellationToken ct = default)
    => await _repository.FindAsync(id, ct);

// ✅ ConfigureAwait(false) in library/infrastructure code
var data = await _httpClient.GetStringAsync(url).ConfigureAwait(false);

// ✅ Return Task directly when just delegating (no await needed)
public Task DeleteAsync(Guid id, CancellationToken ct = default)
    => _repository.DeleteAsync(id, ct);

// ❌ Never use async void (except for event handlers — and even then, wrap it)
public async void DoWork() { ... } // ❌ kills your app silently

// ❌ Never .Result or .Wait() — DEADLOCK WAITING TO HAPPEN
var order = GetOrderAsync(id).Result; // ❌
```

### ValueTask usage

```csharp
// ✅ Use ValueTask<T> ONLY when the result is frequently synchronous
// (e.g., cache hit scenarios). Not everywhere.
public ValueTask<Order?> GetFromCacheAsync(Guid id)
{
    if (_cache.TryGetValue(id, out var order))
        return ValueTask.FromResult<Order?>(order);
    return new ValueTask<Order?>(LoadFromDbAsync(id));
}
```

---

## 5. Error Handling

### Domain exceptions over generic ones

```csharp
// ✅ Define domain-specific exceptions
public sealed class NotFoundException(string message) : Exception(message);
public sealed class ValidationException(string message) : Exception(message);
public sealed class DomainException(string message) : Exception(message);

// ✅ Throw at the right abstraction level
public async Task<Order> GetOrderAsync(Guid id, CancellationToken ct = default)
{
    return await _repository.FindByIdAsync(id, ct)
        ?? throw new NotFoundException($"Order '{id}' does not exist.");
}
```

### Result pattern for expected failures

```csharp
// ✅ Use Result<T> for operations where failure is part of the domain
public record Result<T>(T? Value, string? Error, bool IsSuccess)
{
    public static Result<T> Ok(T value) => new(value, null, true);
    public static Result<T> Fail(string error) => new(default, error, false);
}

// ❌ Don't use exceptions for flow control
try { ... } catch (SomeException) { return defaultValue; } // ❌ disgusting
```

---

## 6. LINQ

```csharp
// ✅ Method syntax for most cases
var total = orders
    .Where(o => o.Status == OrderStatus.Confirmed)
    .Sum(o => o.Total.Amount);

// ✅ Query syntax only when it genuinely reads better (joins, grouping)
var result =
    from order in orders
    join customer in customers on order.CustomerId equals customer.Id
    select new { order.Id, customer.Name };

// ❌ Never mix query and method syntax in the same expression
// ❌ Never materialize with ToList() prematurely — understand deferred execution
// ❌ Never call .Count() when you only need .Any()
if (orders.Count() > 0) // ❌
if (orders.Any())       // ✅
```

---

## 7. Comments & Documentation

### When to comment

```csharp
// ✅ Comment WHY, never WHAT
// The order cannot be cancelled after the payment has been captured by the gateway.
// Attempting to do so requires a refund flow instead — see RefundService.
if (order.Status == OrderStatus.PaymentCaptured)
    throw new DomainException("Cannot cancel: payment already captured.");

// ❌ Useless comment
// Increment counter
counter++;
```

### XML docs on public APIs

```csharp
/// <summary>
/// Places a new order for the specified customer.
/// </summary>
/// <param name="command">The command containing order details.</param>
/// <param name="ct">Cancellation token.</param>
/// <returns>The ID of the newly created order.</returns>
/// <exception cref="NotFoundException">Thrown when the customer does not exist.</exception>
public Task<Guid> PlaceOrderAsync(PlaceOrderCommand command, CancellationToken ct = default);
```

---

## 8. Code Smell Checklist

These are red flags. Seeing any of these in a PR means a conversation is needed:

- [ ] Method longer than 30 lines → Extract
- [ ] Class with more than 5 dependencies → God class, rethink
- [ ] `static` everywhere → You're writing procedural code, not OO
- [ ] `public` fields → Encapsulation is not optional
- [ ] Magic numbers/strings → Extract to constants or configuration
- [ ] Nested ternaries → No. Just no.
- [ ] `catch (Exception ex)` swallowing silently → Criminal
- [ ] `Thread.Sleep()` in production code → You've given up
- [ ] Commented-out code → That's what git is for

---

## 9. Formatting

Enforced via `.editorconfig` + Roslyn analyzers. No manual debates.

```ini
# .editorconfig (root)
root = true

[*.cs]
indent_style = space
indent_size = 4
end_of_line = crlf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

# Braces always on new line
csharp_new_line_before_open_brace = all

# var usage
csharp_style_var_for_built_in_types = false:suggestion
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_elsewhere = false:suggestion
```

### Brace style

```csharp
// ✅ Allman style (braces on their own line) for classes and methods
public class OrderService
{
    public async Task<Order> GetAsync(Guid id, CancellationToken ct)
    {
        // ...
    }
}

// ✅ Expression body for single-expression members
public string FullName => $"{FirstName} {LastName}";
public Task DeleteAsync(Guid id, CancellationToken ct) => _repo.DeleteAsync(id, ct);
```

---

## 10. Required Project Settings

Every `.csproj` must include:

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <AnalysisMode>All</AnalysisMode>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  <LangVersion>latest</LangVersion>
</PropertyGroup>
```

`TreatWarningsAsErrors = true` is non-negotiable. If your code produces warnings, it doesn't ship.

---

## Changelog

### 1.0.0 — 2026-03-24
- Initial version. Covers naming, file organization, C# 13+ features, async/await,
  error handling, LINQ, comments, formatting, and required project settings.
