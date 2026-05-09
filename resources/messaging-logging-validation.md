# MediatR / Serilog / FluentValidation (application layer)

> **Router**: `SKILL.md`. **~110 lines.** Keywords: `IRequestHandler`, `AddMediatR`, `AddValidatorsFromAssemblyContaining`, `ILogger`, pipeline behaviors.
>
> **ViewModels** call **façades** only; persistence details: **`persistence-freesql.md`** and **`mvvm-bindings.md`**.

## MediatR

Typical uses: commands, orchestration, **notifications**. **Plain list/read paths**: **repository + read façade**, **not** `IMediator`, unless your team standardizes CQRS-style `IRequest` queries.

```csharp
services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(CompositionRoot).Assembly));
```

Handler sketch (repository **inside** the handler):

```csharp
public sealed record CreateOrderCommand(string CustomerId, decimal Total) : IRequest<Guid>;

public sealed class CreateOrderHandler(
    IOrderWriteRepository repo,
    ILogger<CreateOrderHandler> log,
    IValidator<CreateOrderCommand> validator)
    : IRequestHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        await validator.ValidateAndThrowAsync(request, ct);
        var id = await repo.InsertAsync(request, ct);
        log.Information("Order {OrderId}", id);
        return id;
    }
}
```

### Layering — ViewModels

Write path: ViewModel → **IOrderWorkflow**; implementation may **`mediator.Send(...)`**. Read path: **`IOrdersQuery` → repository**.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public sealed partial class OrderEditorViewModel(IOrderWorkflow workflow) : ObservableObject
{
    [RelayCommand]
    private async Task SaveAsync(CancellationToken ct) =>
        await workflow.CreateAsync(CustomerId, Total, ct);
}

public sealed class OrderWorkflow(IMediator mediator) : IOrderWorkflow
{
    public Task<Guid> CreateAsync(string customerId, decimal total, CancellationToken ct) =>
        mediator.Send(new CreateOrderCommand(customerId, total), ct);
}

public sealed class OrdersQuery(IOrderReadRepository repo) : IOrdersQuery
{
    public Task<IReadOnlyList<OrderRow>> GetRowsAsync(CancellationToken ct) =>
        repo.ListRowsAsync(ct);
}
```

Optional **`IPipelineBehavior<,>`** for logging, validation, timings.

## Serilog

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateLogger();

var builder = Host.CreateApplicationBuilder(args);
builder.Logging.ClearProviders();
builder.Logging.AddSerilog(Log.Logger, dispose: true);
try { /* build host + Avalonia */ }
finally { Log.CloseAndFlush(); }
```

## FluentValidation

```csharp
services.AddValidatorsFromAssemblyContaining<CreateOrderCommandValidator>();

public sealed class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Total).GreaterThan(0);
    }
}
```

Use **`ValidateAndThrowAsync`** in handlers—or a pipeline behavior. Map **`ValidationException`** in the **façade** layer to bindable UI messages so views never reference MediatR.

## Ordering checklist

1. Validators live in (or referenced by) a scanned assembly.  
2. **`AddValidatorsFromAssemblyContaining`** registered.  
3. Configure **Serilog** early (before sustained traffic/logging needs).  
4. ViewModels see **DTO/result/error view models**, not raw stack traces.
