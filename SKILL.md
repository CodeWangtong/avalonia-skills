---
name: avalonia
description: >-
  Expert guidance for Avalonia on .NET 10: CommunityToolkit.Mvvm, SukiUI + Material icons,
  Generic Host, repositories + FreeSql, optional MediatR for writes, Serilog, FluentValidation.
  Resources are split into small topical files under resources/ for low-token reads.
version: "3.2"
---

# Avalonia skill — **ROUTER** (read this first)

## AI loading contract (save tokens)

1. Use **this page only** for routing; **do not** load all `resources/*.md` at once.
2. Use the **intent → file** table below; **one or two files per task** (add more only for large full-stack work).
3. Each child file begins with a **Router** block (keywords + ~line count). **Search inside that file** before reading the whole file.

## Standard stack

Details live in topical files; this page stays short.

| # | Topic | Choice |
|---|-------|--------|
| 1 | MVVM toolkit | CommunityToolkit: **`public partial`** + `[ObservableProperty]` |
| 2 | Theme | **SukiUI**: Dark + **`ThemeColor="Blue"`**, `SukiWindow` |
| 3 | Icons | Material.Icons.Avalonia |
| 4 | TFM | **net10** |
| 5 | Hosting | Generic Host, `ConfigureServices` |
| 6 | Mapping | Mapster |
| 7–8 | Data | FreeSql; **SQLite** / **MySQL**; **separate config DB**; **monthly shard** for business data |
| 9 | Messaging | MediatR mainly for **writes/orchestration**; **reads = repositories**, not MediatR |
| 10–11 | Log / validate | Serilog; FluentValidation (handlers or façade) |
| 12 | Session UI state | Store pattern |

---

## Intent → **primary file** (match keywords)

| You need | Open |
|----------|------|
| `ObservableProperty`, `RelayCommand`, Messenger, ViewModel layering, no `IMediator`, façade | **`resources/mvvm-toolkit.md`** (~150 lines) |
| `Binding`, `Converter`, `MultiBinding`, `DataTemplate`, design-time `DataContext`, collections | **`resources/mvvm-bindings.md`** (~450 lines) |
| `Program.cs`, `Host`, `ConfigureServices`, resolving `MainWindow` in `App` | **`resources/hosting-di.md`** (~90 lines) |
| `MediatR`, `IRequestHandler`, Serilog, FluentValidation, pipeline behaviors | **`resources/messaging-logging-validation.md`** (~85 lines) |
| Store / reducer / snapshot / undo | **`resources/store-pattern.md`** (~60 lines) |
| FreeSql, `IFreeSql`, monthly sharding, Mapster, repositories | **`resources/persistence-freesql.md`** (~60 lines) |
| `Selector`, `Style`, `SukiTheme`, `ResourceDictionary`, AXAML animation | **`resources/ui-styling.md`** (~70 lines) |
| Debounce, `CancellationTokenSource`, busy + command | **`resources/ui-motion.md`** (~40 lines) |
| Grid, ListBox, DataGrid, `DataGridTemplateColumn`, `$parent`, Tab, Menu | **`resources/ui-controls-catalog.md`** (~120 lines) |
| `TemplatedControl`, virtualization, compiled-binding perf | **`resources/ui-custom-performance.md`** (~40 lines) |
| Packaging, WASM, Android, paths | **`resources/platform-specific.md`** |
| Human-readable index of files | **`DOCUMENTATION.md`** |

### Combo examples (still few files)

- **Shell + DI**: `hosting-di.md`; add `messaging-logging-validation.md` for logging/commands.
- **List + queries**: `mvvm-toolkit.md` (façade) + `persistence-freesql.md` (repos).
- **Theme + layout**: `ui-styling.md` + `ui-controls-catalog.md`.

## Minimal `App.axaml` (copy-paste)

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:suki="https://github.com/kikipoulet/SukiUI"
             xmlns:materialIcons="clr-namespace:Material.Icons.Avalonia;assembly=Material.Icons.Avalonia"
             RequestedThemeVariant="Dark"
             x:Class="MyApp.App">
    <Application.Styles>
        <suki:SukiTheme ThemeColor="Blue" />
        <materialIcons:MaterialIconStyles />
    </Application.Styles>
</Application>
```

## Suggested layout

```
MyApp/
  Views/  ViewModels/  Services/  Stores/  Program.cs  App.axaml
```

---

**Note**: Resources are split for **cheap single-shot reads**. If Avalonia/APIs drift, prefer your **current NuGet + template**.
