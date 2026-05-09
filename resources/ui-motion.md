# UI — motion & debounce (no ReactiveUI)

> **Router**: `SKILL.md`. **~55 lines.** Keywords: `CancellationTokenSource`, `[RelayCommand]`, `IsBusy`.

## Debounce (search/input)

Reuse one **`CancellationTokenSource`** field: on each keystroke **cancel**, **dispose**, **create linked CTS**, **`Task.Delay`**, then invoke façade/search service.

See loading overlay sketch in **`mvvm-bindings.md`**.

```csharp
private CancellationTokenSource? _cts;

// Inject ISearchService as a ViewModel field/ctor parameter.
async Task DebouncedAsync(ISearchService search, string query, CancellationToken appShutdown)
{
    _cts?.Cancel(); _cts?.Dispose();
    _cts = CancellationTokenSource.CreateLinkedTokenSource(appShutdown);
    try {
        await Task.Delay(280, _cts.Token);
        await search.SearchAsync(query, _cts.Token);
    } catch (OperationCanceledException) { /* expected churn */ }
}
```

## Busy + **`[RelayCommand]`**

```csharp
public partial class SearchViewModel : ObservableObject
{
    [ObservableProperty] public partial bool IsBusy { get; set; }
    [ObservableProperty] public partial string Query { get; set; } = "";

    private readonly ISearchService _svc;
    public SearchViewModel(ISearchService svc) => _svc = svc;

    [RelayCommand]
    private async Task SearchAsync(CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(Query)) return;
        IsBusy = true;
        try { await _svc.RunAsync(Query, ct); }
        finally { IsBusy = false; }
    }
}
```

## AXAML motion & transitions

Prefer **`Style.Animations`** / **`Transitions`** ( **`ui-styling.md`** ); avoid heavy code-behind choreography.

## **System.Reactive**

Add only when streams truly need **`CombineLatest` / `Switch`** etc.; **not** in the default stack.
