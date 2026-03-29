# .NET 10 + WPF Architecture Rules

> **Version:** 1.0.0 — 2026-03-24
> Architecture is not decoration. These rules exist because bad structure compounds — every shortcut you take now is debt you pay later with interest. Follow them or justify the deviation in writing.

---

## 1. Solution Structure

### Screaming Architecture — the folder tells you WHAT the system does, not HOW

```
MySolution/
├── src/
│   ├── MyApp.Domain/           # Enterprise business rules — zero dependencies
│   ├── MyApp.Application/      # Use cases — depends only on Domain
│   ├── MyApp.Infrastructure/   # DB, files, APIs, external services
│   └── MyApp.WPF/              # Presentation layer — MVVM, WPF, UI only
├── tests/
│   ├── MyApp.Domain.Tests/
│   ├── MyApp.Application.Tests/
│   └── MyApp.WPF.Tests/
└── MySolution.sln
```

### Layer dependency rule (enforced, not advisory)

```
WPF → Application → Domain
Infrastructure → Application → Domain

Domain has ZERO external dependencies.
Application depends only on Domain abstractions.
Infrastructure implements Application interfaces.
WPF orchestrates everything via DI.
```

**How to enforce it:** Add `<ProjectReference>` only where the dependency is allowed. If you're adding a reference from Domain to anything above it — STOP. You've broken Clean Architecture.

---

## 2. Domain Layer (`MyApp.Domain`)

### What belongs here
- Entities and aggregate roots
- Value objects
- Domain events
- Repository interfaces (not implementations)
- Domain services (stateless, pure logic)
- Domain exceptions
- Enumerations

### What NEVER belongs here
- EF Core, Dapper, ADO.NET — any ORM or data access
- `System.IO`, `System.Net`, `HttpClient`
- Logging frameworks
- WPF or any UI references
- `appsettings.json` / configuration

### Aggregate design

```csharp
// ✅ Aggregates protect their own invariants
public sealed class Order : AggregateRoot
{
    private readonly List<OrderLine> _lines = [];

    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    // Factory method — constructor stays private or internal
    public static Order Create(Guid customerId)
    {
        ArgumentException.ThrowIfNullOrEmpty(customerId.ToString());
        var order = new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Draft };
        order.AddDomainEvent(new OrderCreatedEvent(order.Id));
        return order;
    }

    public void AddLine(Guid productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Lines can only be added to draft orders.");

        _lines.Add(OrderLine.Create(productId, quantity, unitPrice));
    }
}
```

### Value objects are immutable, compared by value

```csharp
public record Money(decimal Amount, string Currency)
{
    public static Money Create(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Amount cannot be negative.");
        if (string.IsNullOrWhiteSpace(currency)) throw new DomainException("Currency is required.");
        return new(amount, currency);
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot add {Currency} and {other.Currency}.");
        return this with { Amount = Amount + other.Amount };
    }
}
```

---

## 3. Application Layer (`MyApp.Application`)

### What belongs here
- Use case commands and queries (CQRS)
- Command/query handlers
- Application services
- DTOs and mapping
- Interface definitions for infrastructure (ports)
- Validation logic (FluentValidation)
- Pipeline behaviors (MediatR)

### CQRS structure

```
Application/
  Orders/
    Commands/
      PlaceOrder/
        PlaceOrderCommand.cs
        PlaceOrderCommandHandler.cs
        PlaceOrderCommandValidator.cs
    Queries/
      GetOrderById/
        GetOrderByIdQuery.cs
        GetOrderByIdQueryHandler.cs
        OrderDto.cs
```

### Command handler pattern

```csharp
public record PlaceOrderCommand(Guid CustomerId, IReadOnlyList<OrderLineDto> Lines)
    : IRequest<Guid>;

public sealed class PlaceOrderCommandHandler(
    IOrderRepository orderRepository,
    ICustomerRepository customerRepository,
    IUnitOfWork unitOfWork)
    : IRequestHandler<PlaceOrderCommand, Guid>
{
    public async Task<Guid> Handle(PlaceOrderCommand request, CancellationToken ct)
    {
        var customer = await customerRepository.FindByIdAsync(request.CustomerId, ct)
            ?? throw new NotFoundException($"Customer '{request.CustomerId}' not found.");

        var order = Order.Create(customer.Id);

        foreach (var line in request.Lines)
            order.AddLine(line.ProductId, line.Quantity, Money.Create(line.UnitPrice, "USD"));

        await orderRepository.AddAsync(order, ct);
        await unitOfWork.CommitAsync(ct);

        return order.Id;
    }
}
```

### Application layer depends on abstractions only

```csharp
// ✅ Interface defined in Application
public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(Guid customerId, CancellationToken ct = default);
}

// Implementation lives in Infrastructure. Application never references EF Core directly.
```

---

## 4. Infrastructure Layer (`MyApp.Infrastructure`)

### What belongs here
- EF Core `DbContext` and configurations
- Repository implementations
- External API clients
- File system access
- Email, SMS, push notification services
- Caching implementations

### Repository implementation

```csharp
public sealed class OrderRepository(AppDbContext dbContext) : IOrderRepository
{
    public async Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default)
        => await dbContext.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct = default)
        => await dbContext.Orders.AddAsync(order, ct);

    public async Task<IReadOnlyList<Order>> GetByCustomerAsync(Guid customerId, CancellationToken ct = default)
        => await dbContext.Orders
            .Where(o => o.CustomerId == customerId)
            .ToListAsync(ct);
}
```

### EF Core configuration — Fluent API only, no DataAnnotations in Domain

```csharp
// ✅ Separate configuration class per entity
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Status).HasConversion<string>().IsRequired();

        builder.OwnsOne(o => o.Total, money =>
        {
            money.Property(m => m.Amount).HasColumnName("TotalAmount").HasPrecision(18, 2);
            money.Property(m => m.Currency).HasColumnName("TotalCurrency").HasMaxLength(3);
        });

        builder.HasMany(o => o.Lines)
            .WithOne()
            .HasForeignKey("OrderId")
            .OnDelete(DeleteBehavior.Cascade);
    }
}

// ❌ Never add [Column], [Required], [MaxLength] to Domain entities
```

---

## 5. WPF Presentation Layer (`MyApp.WPF`)

### MVVM — non-negotiable

WPF without MVVM is not WPF, it's a Windows Forms app with XML. Every UI interaction routes through a ViewModel.

```
WPF/
  Views/
    Orders/
      OrderListView.xaml
      OrderListView.xaml.cs    ← code-behind: only UI logic, zero business logic
      OrderDetailView.xaml
  ViewModels/
    Orders/
      OrderListViewModel.cs
      OrderDetailViewModel.cs
  Controls/                    ← reusable custom controls
  Converters/
  Behaviors/
  App.xaml
  App.xaml.cs
```

### ViewModel rules

```csharp
// ✅ Use CommunityToolkit.Mvvm — no manual INotifyPropertyChanged boilerplate
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public sealed partial class OrderListViewModel : ObservableObject
{
    private readonly IMediator _mediator;

    [ObservableProperty]
    private ObservableCollection<OrderDto> _orders = [];

    [ObservableProperty]
    private bool _isLoading;

    [ObservableProperty]
    private string? _errorMessage;

    public OrderListViewModel(IMediator mediator)
    {
        _mediator = mediator;
    }

    [RelayCommand]
    private async Task LoadOrdersAsync(CancellationToken ct)
    {
        IsLoading = true;
        ErrorMessage = null;

        try
        {
            var result = await _mediator.Send(new GetOrdersQuery(), ct);
            Orders = new ObservableCollection<OrderDto>(result);
        }
        catch (Exception ex)
        {
            ErrorMessage = ex.Message;
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

### View code-behind rules

```csharp
// ✅ Code-behind ONLY for things that are genuinely UI concerns:
//    - Focus management
//    - Animation triggers
//    - DataContext assignment (if not using DI-aware View locator)
public partial class OrderListView : UserControl
{
    public OrderListView(OrderListViewModel viewModel)
    {
        InitializeComponent();
        DataContext = viewModel;
    }
}

// ❌ NEVER in code-behind:
//    - Business logic
//    - Direct service calls
//    - Any logic that could be tested without a UI
```

### Navigation

Use a navigation service abstraction — never navigate from ViewModels using WPF types directly:

```csharp
// ✅ Application layer interface
public interface INavigationService
{
    void NavigateTo<TViewModel>() where TViewModel : ObservableObject;
    void GoBack();
}

// ❌ Never reference Window, Frame, NavigationService directly in ViewModels
```

### Commands

```csharp
// ✅ Use [RelayCommand] from CommunityToolkit.Mvvm
// ✅ Use async commands for any I/O-bound operation
// ✅ Pass CancellationToken to commands that do async work
// ❌ Never block the UI thread with .Result or .Wait()
// ❌ Never use Task.Run for CPU-bound work in a desktop app without a loading state
```

---

## 6. Dependency Injection

### Registration — organized by layer

```csharp
// Each layer exposes its own extension method
// Domain has no services to register (pure logic)

// Application/DependencyInjection.cs
public static class ApplicationServiceCollectionExtensions
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(PlaceOrderCommand).Assembly));
        services.AddValidatorsFromAssembly(typeof(PlaceOrderCommand).Assembly);
        return services;
    }
}

// Infrastructure/DependencyInjection.cs
public static class InfrastructureServiceCollectionExtensions
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(opt =>
            opt.UseSqlServer(configuration.GetConnectionString("Default")));
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        return services;
    }
}

// WPF App.xaml.cs
protected override void OnStartup(StartupEventArgs e)
{
    var services = new ServiceCollection();
    services.AddApplication();
    services.AddInfrastructure(Configuration);
    services.AddSingleton<MainWindow>();
    // Register ViewModels as Transient
    services.AddTransient<OrderListViewModel>();

    _serviceProvider = services.BuildServiceProvider();
    _serviceProvider.GetRequiredService<MainWindow>().Show();
}
```

### Lifetime rules

| Type | Lifetime |
|---|---|
| `DbContext` | `Scoped` (per operation scope) |
| Repositories | `Scoped` |
| Application services | `Scoped` |
| ViewModels | `Transient` |
| Navigation service | `Singleton` |
| Http clients | Via `IHttpClientFactory` |

---

## 7. Configuration

```csharp
// ✅ Strongly-typed configuration — no magic strings in code
public sealed record DatabaseOptions
{
    public const string Section = "Database";
    public required string ConnectionString { get; init; }
    public int CommandTimeout { get; init; } = 30;
}

// Registration
services.AddOptions<DatabaseOptions>()
    .BindConfiguration(DatabaseOptions.Section)
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Usage via injection
public class SomeService(IOptions<DatabaseOptions> options) { ... }

// ❌ Never use IConfiguration directly in services — inject IOptions<T>
// ❌ Never hardcode connection strings or credentials
```

---

## 8. Logging

```csharp
// ✅ Use ILogger<T> from Microsoft.Extensions.Logging — always
// ✅ Use structured logging with named parameters
_logger.LogInformation("Order {OrderId} placed for customer {CustomerId}.", order.Id, order.CustomerId);

// ✅ Use LoggerMessage source generator for hot paths (performance)
public static partial class OrderLogMessages
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Processing order {OrderId}")]
    public static partial void LogProcessingOrder(this ILogger logger, Guid orderId);
}

// ❌ Never log sensitive data (PII, passwords, tokens)
// ❌ Never use Console.WriteLine in production code
// ❌ Never swallow exceptions without logging them
```

---

## 9. Testing Standards

### Test project structure mirrors source

```
MyApp.Application.Tests/
  Orders/
    Commands/
      PlaceOrderCommandHandlerTests.cs
    Queries/
      GetOrderByIdQueryHandlerTests.cs
```

### Naming convention

```
MethodName_StateUnderTest_ExpectedBehavior
PlaceOrder_WhenCustomerDoesNotExist_ThrowsNotFoundException
GetOrder_WhenOrderExists_ReturnsOrderDto
```

### Use cases test Application layer, not Infrastructure

```csharp
// ✅ Mock the repository — test behavior, not EF Core
public class PlaceOrderCommandHandlerTests
{
    private readonly Mock<IOrderRepository> _orderRepo = new();
    private readonly Mock<ICustomerRepository> _customerRepo = new();
    private readonly Mock<IUnitOfWork> _unitOfWork = new();

    [Fact]
    public async Task Handle_WhenCustomerNotFound_ThrowsNotFoundException()
    {
        _customerRepo.Setup(r => r.FindByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync((Customer?)null);

        var handler = new PlaceOrderCommandHandler(_orderRepo.Object, _customerRepo.Object, _unitOfWork.Object);
        var command = new PlaceOrderCommand(Guid.NewGuid(), []);

        await Assert.ThrowsAsync<NotFoundException>(() => handler.Handle(command, CancellationToken.None));
    }
}
```

### WPF ViewModel tests

ViewModels must be testable WITHOUT a UI. If you can't test a ViewModel in a plain xUnit project, your ViewModel is broken:

```csharp
[Fact]
public async Task LoadOrders_WhenQuerySucceeds_PopulatesOrders()
{
    var mediator = new Mock<IMediator>();
    mediator.Setup(m => m.Send(It.IsAny<GetOrdersQuery>(), It.IsAny<CancellationToken>()))
        .ReturnsAsync(new List<OrderDto> { new(Guid.NewGuid(), "ACME", 100m) });

    var vm = new OrderListViewModel(mediator.Object);
    await vm.LoadOrdersCommand.ExecuteAsync(null);

    Assert.Single(vm.Orders);
    Assert.False(vm.IsLoading);
}
```

---

## 10. Architecture Fitness Functions

Run these checks in CI. If any fails, the build fails:

| Check | Tool |
|---|---|
| Layer dependency violations | ArchUnitNET |
| Public APIs without nullable enabled | Roslyn analyzers |
| Missing XML docs on public types | `CS1591` warning-as-error |
| TODO/FIXME in production code | Custom Roslyn analyzer or grep in CI |

### ArchUnitNET example

```csharp
// Enforce that Domain has no dependency on Infrastructure
[Fact]
public void Domain_Should_Not_Reference_Infrastructure()
{
    var domain = ArchRuleDefinition.Types()
        .That().ResideInNamespace("MyApp.Domain.*")
        .Should().NotDependOnAny(
            ArchRuleDefinition.Types().That().ResideInNamespace("MyApp.Infrastructure.*"));

    domain.Check(Architecture);
}
```

---

## Quick Reference Card

| Situation | Rule |
|---|---|
| New feature | Command + Handler + Validator + Tests TOGETHER |
| UI interaction | ViewModel command, never code-behind logic |
| DB access | Repository interface in Application, implementation in Infrastructure |
| Cross-cutting concern | MediatR pipeline behavior, not base class |
| Configuration value | `IOptions<T>`, never raw `IConfiguration` |
| Error in use case | Domain exception or Result<T>, never return null |
| Async operation | Always `CancellationToken`, never `.Result` |
| New dependency | Constructor injection only, never service locator |

---

## Changelog

### 1.0.0 — 2026-03-24
- Initial version. Covers solution structure, Clean Architecture layers, Domain design
  (aggregates, value objects), Application CQRS pattern, Infrastructure (EF Core Fluent API),
  WPF MVVM with CommunityToolkit.Mvvm, DI lifetime rules, configuration, logging,
  testing standards, and architecture fitness functions.
