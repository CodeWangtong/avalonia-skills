# Generic Host & DI (Avalonia)

> **Router**: root `SKILL.md`. **~110 lines.** Keywords: `Host.CreateApplicationBuilder`, `ConfigureServices`, `Program.Services`, `IClassicDesktopStyleApplicationLifetime`.
>
> ViewModels **must not** take `IMediator` directly (see `mvvm-toolkit.md`).

Compose with **`Microsoft.Extensions.Hosting`** and share **one** `IServiceProvider` graph with Avalonia **`AppBuilder`**.

## Roles

| Component | Responsibility |
|-----------|----------------|
| **Host.CreateApplicationBuilder** | Configuration + **DI** + logging providers |
| **Avalonia** | `Application` / lifetime (desktop vs mobile) |
| **Extension methods** `AddMyAppServices` | Central registrations |

## `Program.cs` sketch (desktop)

```csharp
using Avalonia;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace MyApp;

public static class Program
{
    public static IServiceProvider Services { get; private set; } = default!;

    public static void Main(string[] args)
    {
        var builder = Host.CreateApplicationBuilder(args);
        builder.Services.AddMyAppServices(builder.Configuration);
        using var host = builder.Build();
        Services = host.Services;
        BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);
    }

    private static AppBuilder BuildAvaloniaApp() =>
        AppBuilder.Configure<App>()
            .UsePlatformDetect()
            .WithInterFont()
            .LogToTrace();
}
```

Match **your official Avalonia template** (some merge `host.Run` with Avalonia lifetime). Invariant: **`Services`** comes only from **`host.Build()`**, and **`host.Dispose()`** runs after Avalonia shuts down.

## `ConfigureServices` sketch

```csharp
using FluentValidation;
using MediatR;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace Microsoft.Extensions.DependencyInjection;

public static class CompositionRoot
{
    public static IServiceCollection AddMyAppServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddSingleton<IConfiguration>(configuration);
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(CompositionRoot).Assembly));
        services.AddValidatorsFromAssemblyContaining(typeof(CompositionRoot));
        // services.AddScoped<IOrdersQuery, OrdersQuery>();
        // services.AddScoped<IOrderReadRepository, OrderReadRepository>();
        // FreeSql: see `persistence-freesql.md`
        services.AddTransient<MainViewModel>();
        services.AddSingleton<MainWindow>();
        return services;
    }
}
```

Serilog: wire during host startup per **`messaging-logging-validation.md`**.

## `App.axaml.cs` (resolve main window)

```csharp
using Avalonia;
using Avalonia.Controls.ApplicationLifetimes;
using Microsoft.Extensions.DependencyInjection;
using MyApp.Views;

namespace MyApp;

public partial class App : Application
{
    public override void OnFrameworkInitializationCompleted()
    {
        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
            desktop.MainWindow = Program.Services.GetRequiredService<MainWindow>();
        base.OnFrameworkInitializationCompleted();
    }
}
```

## Configuration

`appsettings.json` + environment overlays; secrets via User Secrets or OS vault.

## Checklist

- [ ] **Single** DI graph per process (no second ad-hoc `BuildServiceProvider`)
- [ ] ViewModels usually **Transient**
- [ ] façade + repos + FreeSql: **`persistence-freesql.md`**
- [ ] MediatR / Serilog / FluentValidation: **`messaging-logging-validation.md`**
