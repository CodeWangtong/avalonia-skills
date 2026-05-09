# UI — control cheat sheet (layout / inputs / lists)

> **Router**: `SKILL.md`. **~120 lines.** Abbr. Keywords: `Grid`, `ItemsControl`, `ListBox`, `DataGrid`, `DataGridTemplateColumn`, compiled binding, `$parent`.
>
> Full property reference: [Avalonia Controls](https://docs.avaloniaui.net/docs/guides/basics/).

## Stack policy

With **SukiUI** and **compiled bindings**: set **`x:DataType`** on roots (see **`mvvm-toolkit.md`**).

---

## Layout

**Grid**: `ColumnDefinitions="Auto,*"`, `RowDefinitions="Auto,*"`, `Grid.ColumnSpan`.  
**StackPanel**: `Orientation`, `Spacing`.  
**DockPanel**: `DockPanel.Dock`.  
**ScrollViewer**: `HorizontalScrollBarVisibility`.

```xml
<Grid ColumnDefinitions="240,*" RowDefinitions="Auto,*">
    <StackPanel Grid.ColumnSpan="2" Orientation="Horizontal" Spacing="8">
        <TextBox Width="240" Watermark="Filter…"/>
        <Button Content="Search"/>
    </StackPanel>
    <ListBox Grid.Row="1" ItemsSource="{Binding Items}"/>
</Grid>
```

---

## Input

```xml
<TextBox Text="{Binding Name}"/>
<NumericUpDown Value="{Binding Qty}" Minimum="0"/>
<Slider Minimum="0" Maximum="100" Value="{Binding Volume}"/>
<DatePicker SelectedDate="{Binding Due}"/>
<ComboBox ItemsSource="{Binding Choices}" SelectedItem="{Binding Current}"/>
```

---

## Lists & grid

```xml
<ListBox ItemsSource="{Binding Users}"
         SelectedItem="{Binding Selected}">
    <ListBox.ItemTemplate>
        <DataTemplate x:DataType="models:User">
            <StackPanel><TextBlock Text="{Binding DisplayName}"/></StackPanel>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>

<DataGrid ItemsSource="{Binding Rows}"
          AutoGenerateColumns="False">
    <DataGrid.Columns>
        <DataGridTextColumn Header="Title" Binding="{Binding Title}" />
        <DataGridCheckBoxColumn Header="Done" Binding="{Binding Done}" />
    </DataGrid.Columns>
</DataGrid>
```

### `DataGridTemplateColumn`: row buttons bound to outer ViewModel

Inside `ItemTemplate` / `CellTemplate`, **`DataContext` is the row model**. If `Command` lives on the **root `UserControl` ViewModel**, use **`$parent[UserControl]`** to reach outer `DataContext`; `CommandParameter` can still pass the **row** with `{Binding}`.

`Classes` (e.g. `secondary-action` / `primary-action`) should match styles in your solution or SukiUI.

```xml
<DataGridTemplateColumn Header="Actions" IsReadOnly="True">
    <DataGridTemplateColumn.CellTemplate>
        <DataTemplate x:DataType="models:PlcRegisterEditorItem">
            <StackPanel Orientation="Horizontal" Spacing="8">
                <Button
                    Classes="secondary-action"
                    Command="{Binding $parent[UserControl].DataContext.ReadCommand}"
                    CommandParameter="{Binding}"
                    Content="Read" />
                <Button
                    Classes="primary-action"
                    Command="{Binding $parent[UserControl].DataContext.WriteCommand}"
                    CommandParameter="{Binding}"
                    Content="Write"
                    IsEnabled="{Binding CanWrite}" />
            </StackPanel>
        </DataTemplate>
    </DataGridTemplateColumn.CellTemplate>
</DataGridTemplateColumn>
```

---

## Navigation & chrome

```xml
<TabControl SelectedIndex="{Binding TabIndex}">
    <TabItem Header="Home"><ContentControl /></TabItem>
</TabControl>

<Menu>
    <MenuItem Header="_File"><MenuItem Header="Quit"/></MenuItem>
</Menu>
```

---

## Window / dialogs

Flyout, `Window.ShowDialog`, `ContentDialog`, etc., depend on your **Avalonia version and head project**; follow your repo’s **desktop template** first.
