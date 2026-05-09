# Store pattern (snapshots / undo-friendly state)

> **Router**: `SKILL.md`. **~80 lines.** Keywords: `Reducer`, `Dispatch`, immutable `State`, `Changed`.
>
> Coexists with FreeSql entities: Store suits **session/editor UI snapshots** vs durable DB—see **`persistence-freesql.md`**.

## Concepts

| Concept | Meaning |
|---------|---------|
| **State** | Immutable snapshot at a point in time |
| **Action** | Describes what happened |
| **Reducer** | Pure `(State, Action) => State` |
| **Store** | Holds State, exposes `Dispatch`, notifies subscribers |

## Minimal implementation

```csharp
namespace MyApp.Stores;

public interface IAppStore<TState> where TState : class
{
    TState State { get; }
    event Action<TState>? Changed;
    void Dispatch(IAction action);
}

public interface IAction { }

public sealed class SimpleStore<TState> : IAppStore<TState> where TState : class
{
    private readonly object _gate = new();
    private readonly Func<TState, IAction, TState> _reducer;
    private TState _state;

    public SimpleStore(TState initial, Func<TState, IAction, TState> reducer)
    {
        _state = initial;
        _reducer = reducer;
    }

    public TState State
    {
        get { lock (_gate) { return _state; } }
    }

    public event Action<TState>? Changed;

    public void Dispatch(IAction action)
    {
        TState next;
        lock (_gate)
        {
            next = _reducer(_state, action);
            if (ReferenceEquals(next, _state)) return;
            _state = next;
        }
        Changed?.Invoke(next);
    }
}
```

## Avalonia integration

- Subscribe to **`Changed`** and mirror fields into ViewModels (`[ObservableProperty]`).
- **`Dispatch` on UI thread** when state drives UI; after async work finishes: `Dispatcher.UIThread.Post(() => store.Dispatch(...))`.

## Snapshot serialization

```csharp
JsonSerializer.Serialize(state); // optional MessagePack; config DB/small files—not bulk sharded DB
```

## Undo / redo

Store **inverse actions** or **previous State**; for large graphs use **patch/diff**.

## When **not** to use

Simple CRUD/forms: **`ViewModel + FluentValidation`** is enough.
