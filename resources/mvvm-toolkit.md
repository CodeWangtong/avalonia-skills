# MVVM — Community Toolkit

> Router: root `SKILL.md`. **~150 lines.** Keywords: `ObservableProperty`, `RelayCommand`, Messenger, façade, repository, compiled binding intro.

## Community Toolkit (Mvvm)

Primary MVVM toolkit for this skill. It reduces boilerplate with **Roslyn source generators** and aligns with **`INotifyPropertyChanged`** / **`ICommand`** expectations in Avalonia bindings.

NuGet:

```xml
<PackageReference Include="CommunityToolkit.Mvvm" Version="8.*" />
```

(Use latest **8.x** compatible with net10.)

## ObservableObject

Base type providing `OnPropertyChanging` / `OnPropertyChanged` and batching helpers.

```csharp
public partial class MyVm : ObservableObject
{
    // Prefer attributes + generated properties (below).
}
```

## ObservableProperty

This skill prefers **`[ObservableProperty]` on `public partial` auto-properties** (CommunityToolkit.Mvvm 8.x+). The generator supplies the backing field and wires **`INotifyPropertyChanged`**.

### Preferred style (project convention)

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

public partial class ChartViewModel : ObservableObject
{
    [ObservableProperty] public partial double TrendLabelStep { get; set; } = 1;
}
```

- Property name in XAML/bindings is **`TrendLabelStep`** (same identifier).
- Class must be **`partial`** so the generator can emit the other part.

### Alternative: private backing field

Older samples use a field; still valid if you prefer it:

```csharp
public partial class ProfileViewModel : ObservableObject
{
    [ObservableProperty] private string _displayName = string.Empty;
    // Generates public DisplayName wrapping _displayName
}
```

Conceptually equivalent to **`SetProperty(ref field, value)`** for field-based generation.

### Options (either syntax)

Apply extra attributes next to **`[ObservableProperty]`**:

```csharp
[ObservableProperty]
[NotifyCanExecuteChangedFor(nameof(SaveCommand))]
[NotifyPropertyChangedFor(nameof(FullName))]
public partial string? First { get; set; }
```

For field-based generation, **`_first`** yields **`First`** unless you override with **`PropertyName`**.

Explore **`CommunityToolkit.Mvvm.ComponentModel`** docs for **`NotifyChangesTo`** and related attributes.

## RelayCommand

```csharp
using CommunityToolkit.Mvvm.Input;

public partial class EditorViewModel : ObservableObject
{
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(SaveCommand))]
    public partial string Title { get; set; } = string.Empty;

    [RelayCommand(CanExecute = nameof(CanSave))]
    private void Save() { /* ... */ }

    private bool CanSave() => !string.IsNullOrWhiteSpace(Title);
}
```

### Async commands

```csharp
[RelayCommand]
private async Task RefreshAsync(CancellationToken cancellationToken)
{
    await _service.LoadAsync(cancellationToken);
}
```

Bindings stay the same: **`Command="{Binding RefreshCommand}"`**. The toolkit wraps exceptions; log failures in the method or a global handler.

### Cancellation

Pass **`CancellationToken`** to async relay methods when canceling navigation or closing tabs; cancel when the command is superseded if you implement custom semantics.

## ObservableRecipient & messages

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Messaging;
using CommunityToolkit.Mvvm.Messaging.Messages;

public partial class ShellViewModel : ObservableRecipient
{
    public ShellViewModel()
    {
        Messenger.Register<ShellViewModel, LogoutMessage>(this, (_, _) => OnLogout());
    }

    private void OnLogout() { /* ... */ }
}

public sealed class LogoutMessage { }
```

For **global** vs **recipient** scope, pick constructor overloads on `Messenger.Register`. Prefer **weak** references by default to avoid leaks with long-lived shells.

## Messenger (static)

`WeakReferenceMessenger.Default` is convenient for app-wide events (theme changes, culture changes). Keep payloads **small** and **immutable**.

## INotifyPropertyChanged attributes (legacy path)

`[INotifyPropertyChanged]` on a `partial` class generates `INotifyPropertyChanged` implementation without inheriting `ObservableObject`—useful for constrained inheritance hierarchies.

## Collections

- **`ObservableCollection<T>`** for item panels bound to ItemsControls / ListBox.
- For bulk updates, consider **`ObservableCollection`** replacement rather than item-by-item if perf matters (batch reset bound collection).

CommunityToolkit provides **`ObservableObject`** helpers like **`SetProperty`** when hand-writing properties without generators.

## compiled bindings (`x:DataType`)

Always align AXAML root `x:DataType` with ViewModel type so Avalonia resolves commands/properties at compile time:

```xml
<UserControl xmlns:vm="using:MyApp.ViewModels"
             x:DataType="vm:ProfileViewModel">
    <TextBox Text="{Binding DisplayName}" />
</UserControl>
```

## Business layer (no `IMediator` in ViewModels)

**ViewModels do not inject `IMediator` or call `Send`.** Behaviors live behind a **business / application API** (`IOrdersQuery`, `IOrderWorkflow`, `ISettingsProvider`, …). **`Data reads` go through that API into database repositories (or read services)** — not through MediatR. **MediatR** (when used) sits on **command / workflow / notification** paths and still typically ends in repositories or domain services—not in place of repositories for straightforward queries.

```csharp
// ViewModel — depends on application contract only
public partial class OrdersViewModel(IOrdersQuery ordersQuery) : ObservableObject
{
    [ObservableProperty]
    public partial ObservableCollection<OrderRow> Orders { get; set; } = new();

    [RelayCommand]
    private async Task LoadAsync(CancellationToken ct)
    {
        var rows = await ordersQuery.GetRowsAsync(ct);
        Orders = new ObservableCollection<OrderRow>(rows);
    }
}
```

```csharp
// Application/read façade — delegates to persistence, not IMediator
public sealed class OrdersQuery(IOrderReadRepository orders) : IOrdersQuery
{
    public Task<IReadOnlyList<OrderRow>> GetRowsAsync(CancellationToken ct) =>
        orders.ListRowsAsync(ct);
}
```

Register **`IOrdersQuery` → `OrdersQuery`** and **`IOrderReadRepository` → concrete repository** (FreeSql-backed) in **`ConfigureServices`** (**`hosting-di.md`**). Read model: **`persistence-freesql.md`**; commands & validation: **`messaging-logging-validation.md`**.

## Testing

ViewModels remain **framework-agnostic** aside from **`ObservableObject`**:

```csharp
// Assuming: [ObservableProperty] public partial string DisplayName { get; set; }
var vm = new ProfileViewModel(fakeService);
vm.DisplayName = "Test";
Assert.Equal("Test", vm.DisplayName);
```

Commands: cast **`IRelayCommand`** and inspect **`CanExecute`**.

## Pitfalls

1. **`partial`** — the **class** must be **`partial`**; **`public partial`** properties require **`partial`** on the property declaration.
2. **Naming** — with **partial properties**, bindings use the **same name** as the declared property (**`TrendLabelStep`**, etc.). With **fields**, **`_camelCase`** maps to **`PascalCase`** generated surface.
3. **Heavy work in getters** — avoid; getters should be side-effect free.
4. **Circular `NotifyPropertyChangedFor`** — can cause update storms; simplify graph.
5. **Leak** — unregister **Messenger** subscriptions when view models end (especially non-weak patterns).

## Further reading in this skill

- **`mvvm-bindings.md`** — binding modes, converters, collections, design-time data.
- **`hosting-di.md`** / **`messaging-logging-validation.md`** / **`persistence-freesql.md`** — host, cross-cutting, repositories; **Toolkit Messenger** still for lightweight UI events.

