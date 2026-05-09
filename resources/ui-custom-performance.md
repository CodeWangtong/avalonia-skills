# UI — custom controls & performance

> **Router**: `SKILL.md`. **~55 lines.** Keywords: `TemplatedControl`, `Simple` virtualization, compiled bindings project flag.

## TemplatedControl skeleton

```csharp
public class Badge : TemplatedControl
{
    public static readonly StyledProperty<int> CountProperty =
        AvaloniaProperty.Register<Badge, int>(nameof(Count));

    public int Count
    {
        get => GetValue(CountProperty);
        set => SetValue(CountProperty, value);
    }
}
```

Supply **ControlTemplate** in **Generic.xaml / Themes**; align **`PART_`** names with `OnApplyTemplate`.

---

## Virtualization (large lists)

```xml
<ListBox ItemsSource="{Binding Large}"
         VirtualizationMode="Simple">
    <ListBox.ItemTemplate>...</ListBox.ItemTemplate>
</ListBox>
```

Paging / **incremental load**: append to `ObservableCollection` in the ViewModel instead of replacing huge lists at once (pattern near **`mvvm-bindings.md`**).

---

## Compiled bindings

Enable **`AvaloniaUseCompiledBindingsByDefault`** and set AXAML **`x:DataType`** — see **`mvvm-toolkit.md`**.

---

## Performance checklist

| Technique | Why |
|-----------|-----|
| `x:DataType` | Faster binding resolution |
| Virtualized list | `VirtualizationMode` |
| Debounce | **`ui-motion.md`** |
| Fewer `PropertyChanged` storms | Batch collection replacement or diff |
