# MVVM — Avalonia bindings & converters

> Router: root `SKILL.md`. **~450 lines.** Keywords: Binding, Converter, ObservableCollection, DataTemplate, Generic Host snippet, master-detail.

## Avalonia bindings & patterns

Core patterns for **Model-View-ViewModel** and data binding in Avalonia.

**Project standard**: **CommunityToolkit.Mvvm** for ViewModels; **Generic Host** for DI; **business/application façades** into ViewModels (**no `IMediator`**). **Data reads** go **façade → repository**; **writes** may use **MediatR handlers** (validators + repository) behind a workflow façade. **Mapster** at DTO/entity edges. Do **not** default to ReactiveUI unless the repository explicitly adds it.

## MVVM Architecture

### Overview

Model-View-ViewModel (MVVM) separates concerns into three layers:

- **Model**: Business logic and data
- **View**: UI presentation (XAML)
- **ViewModel**: Bridge between View and Model, handles state and commands

### Project Structure

```
MyAvaloniaApp/
├── Models/                     # Business logic and data
│   ├── User.cs
│   ├── Product.cs
│   └── IDataService.cs        # Service interfaces
├── ViewModels/                # MVVM logic
│   ├── MainViewModel.cs
│   ├── UserListViewModel.cs
│   └── ViewModelBase.cs       # Common base class
├── Views/                     # XAML views
│   ├── MainWindow.axaml
│   ├── UserListView.axaml
│   └── ...
└── Services/                  # Application services
    ├── DataService.cs
    ├── NavigationService.cs
    └── ...
```

### ViewModel base (CommunityToolkit.Mvvm)

Prefer **`ObservableObject`** with **`[ObservableProperty]`** and **`[RelayCommand]`**. Use **`public partial`** auto-properties (same name surfaces to bindings):

`[ObservableProperty] public partial double TrendLabelStep { get; set; } = 1;`

See preceding **ObservableProperty** section for private backing-field variant.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

public partial class MainViewModel : ObservableObject
{
    [ObservableProperty] public partial string Name { get; set; } = string.Empty;
    [ObservableProperty] public partial string Email { get; set; } = string.Empty;
    [ObservableProperty] public partial ObservableCollection<User> Users { get; set; } = new();

    public MainViewModel()
    {
        // Inject application façades (IOrderWorkflow, IUserDirectory, …) — never IMediator / IFreeSql here.
    }

    [RelayCommand]
    private void Save()
    {
        // Save via injected business façade, e.g. await _workflow.SaveAsync(...)
    }

    [RelayCommand]
    private void Load()
    {
        // Load logic
    }
}
```

### Manual INotifyPropertyChanged (escape hatch)

Use only when you cannot reference the toolkit in a small shared library:

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

public class MainViewModel : INotifyPropertyChanged
{
    private string _name;
    public string Name
    {
        get => _name;
        set => SetProperty(ref _name, value);
    }

    private string _email;
    public string Email
    {
        get => _email;
        set => SetProperty(ref _email, value);
    }

    public event PropertyChangedEventHandler PropertyChanged;

    protected void SetProperty<T>(ref T field, T value, [CallerMemberName] string propertyName = "")
    {
        if (!Equals(field, value))
        {
            field = value;
            OnPropertyChanged(propertyName);
        }
    }

    protected void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

## Data Binding Fundamentals

### Binding Modes

```xml
<!-- OneWay: View updates when ViewModel changes (default for TextBlock) -->
<TextBlock Text="{Binding Name}" />

<!-- TwoWay: View and ViewModel sync bidirectionally (default for TextBox) -->
<TextBox Text="{Binding Name, Mode=TwoWay}" />

<!-- OneTime: Bind once at initialization, no updates -->
<TextBlock Text="{Binding Name, Mode=OneTime}" />

<!-- OneWayToSource: ViewModel updates when View changes -->
<Slider Value="{Binding Volume, Mode=OneWayToSource}" />
```

### Binding Paths

```xml
<!-- Simple property binding -->
<TextBlock Text="{Binding Name}" />

<!-- Nested property binding -->
<TextBlock Text="{Binding User.Name}" />

<!-- Collection indexing -->
<TextBlock Text="{Binding Items[0].Name}" />

<!-- Binding to parent DataContext -->
<TextBlock Text="{Binding Path=DataContext.Title, RelativeSource={RelativeSource AncestorType=Window}}" />

<!-- Self binding -->
<Button Content="{Binding Path=(Button.Content), RelativeSource={RelativeSource Self}}" />
```

### Binding to Commands

```xml
<!-- Basic command -->
<Button Content="Save" Command="{Binding SaveCommand}" />

<!-- Command with parameter -->
<Button Content="Delete" 
        Command="{Binding DeleteCommand}"
        CommandParameter="{Binding SelectedItem}" />

<!-- Multi-binding to command -->
<Button Content="Search">
    <Button.Command>
        <MultiBinding>
            <Binding Path="SearchCommand" />
            <Binding Path="SearchText" />
            <Binding Path="SearchCategory" />
        </MultiBinding>
    </Button.Command>
</Button>
```

### Multi-Binding

```xml
<!-- Combine multiple bindings -->
<TextBlock>
    <TextBlock.Text>
        <MultiBinding StringFormat="{}{0} - {1}">
            <Binding Path="FirstName" />
            <Binding Path="LastName" />
        </MultiBinding>
    </TextBlock.Text>
</TextBlock>

<!-- Multi-binding with converter -->
<TextBlock>
    <TextBlock.Text>
        <MultiBinding Converter="{StaticResource FullAddressConverter}">
            <Binding Path="Street" />
            <Binding Path="City" />
            <Binding Path="State" />
            <Binding Path="ZipCode" />
        </MultiBinding>
    </TextBlock.Text>
</TextBlock>
```

### Binding Validation

```xml
<!-- Validate with bound property -->
<TextBox Text="{Binding Email}">
    <DataValidationErrors.Error>
        <Binding Path="Email" />
    </DataValidationErrors.Error>
</TextBox>

<!-- Display validation errors -->
<TextBlock Foreground="Red" 
           Text="{Binding (DataValidationErrors.Error)}" />
```

## Value Converters

### Basic Converter

```csharp
using System.Globalization;
using Avalonia.Data.Converters;

public class BoolToVisibilityConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        if (value is bool boolValue)
            return boolValue ? Avalonia.Controls.Visibility.Visible : Avalonia.Controls.Visibility.Collapsed;
        return Avalonia.Controls.Visibility.Collapsed;
    }

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
    {
        if (value is Avalonia.Controls.Visibility visibility)
            return visibility == Avalonia.Controls.Visibility.Visible;
        return false;
    }
}
```

### Multi-Value Converter

```csharp
public class FullNameConverter : IMultiValueConverter
{
    public object Convert(IList<object> values, Type targetType, object parameter, CultureInfo culture)
    {
        if (values.Count < 2) return "";
        var firstName = values[0]?.ToString() ?? "";
        var lastName = values[1]?.ToString() ?? "";
        return $"{firstName} {lastName}".Trim();
    }
}
```

### Using Converters

```xml
<Window.Resources>
    <converters:BoolToVisibilityConverter x:Key="BoolToVisibility" />
    <converters:FullNameConverter x:Key="FullName" />
</Window.Resources>

<!-- Single value converter -->
<TextBlock Text="{Binding Status, Converter={StaticResource StatusToStringConverter}}" />

<!-- Multi-value converter -->
<TextBlock>
    <TextBlock.Text>
        <MultiBinding Converter="{StaticResource FullName}">
            <Binding Path="FirstName" />
            <Binding Path="LastName" />
        </MultiBinding>
    </TextBlock.Text>
</TextBlock>
```

## Dependency injection (Generic Host)

Register services with **`Microsoft.Extensions.Hosting`** (`Host.CreateApplicationBuilder`) and **`ConfigureServices`**. Keep **one** `IServiceProvider` graph for the process.

**Full examples**: `hosting-di.md` (Program / `ConfigureServices`), `messaging-logging-validation.md` (Serilog / MediatR / FV).

Minimal mental model:

```csharp
// Program / composition root (sketch)
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddSingleton<IDataService, DataService>();
builder.Services.AddTransient<MainViewModel>();
builder.Services.AddSingleton<MainWindow>();
var host = builder.Build();
// Bridge host.Services into Avalonia App — see hosting-di.md
```

```csharp
// MainWindow.axaml.cs — constructor injection
public partial class MainWindow : Window
{
    public MainWindow(MainViewModel viewModel)
    {
        InitializeComponent();
        DataContext = viewModel;
    }
}
```

Small prototypes may still use `new ServiceCollection().BuildServiceProvider()`; production code in this skill should prefer **Generic Host**.

## Collections and Binding

### ObservableCollection Binding

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

public partial class UserListViewModel : ObservableObject
{
    [ObservableProperty] public partial ObservableCollection<User> Users { get; set; } = new();
    [ObservableProperty] public partial User? SelectedUser { get; set; }

    public UserListViewModel(IDataService dataService)
    {
        _dataService = dataService;
        LoadUsers();
    }

    private readonly IDataService _dataService;

    private void LoadUsers()
    {
        var users = _dataService.GetAllUsers();
        Users = new ObservableCollection<User>(users);
    }

    public void AddUser(User user) => Users.Add(user);

    public void RemoveUser(User user) => Users.Remove(user);
}
```

### ListBox Binding

```xml
<ListBox ItemsSource="{Binding Users}"
         SelectedItem="{Binding SelectedUser, Mode=TwoWay}"
         SelectionMode="Single">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <StackPanel Orientation="Horizontal" Spacing="10">
                <Image Source="{Binding Avatar}" Width="32" Height="32" />
                <StackPanel>
                    <TextBlock Text="{Binding Name}" FontWeight="Bold" />
                    <TextBlock Text="{Binding Email}" FontSize="11" Foreground="Gray" />
                </StackPanel>
            </StackPanel>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>
```

### DataGrid Binding

```xml
<DataGrid ItemsSource="{Binding Users}"
          SelectedItem="{Binding SelectedUser, Mode=TwoWay}"
          AutoGenerateColumns="False"
          CanUserReorderColumns="True">
    <DataGrid.Columns>
        <DataGridTextColumn Header="Name" Binding="{Binding Name}" />
        <DataGridTextColumn Header="Email" Binding="{Binding Email}" />
        <DataGridCheckBoxColumn Header="Active" Binding="{Binding IsActive}" />
        <DataGridTemplateColumn Header="Actions" Width="100">
            <DataGridTemplateColumn.CellTemplate>
                <DataTemplate>
                    <StackPanel Orientation="Horizontal" Spacing="5">
                        <Button Content="Edit" Command="{Binding EditCommand}" />
                        <Button Content="Delete" Command="{Binding DeleteCommand}" />
                    </StackPanel>
                </DataTemplate>
            </DataGridTemplateColumn.CellTemplate>
        </DataGridTemplateColumn>
    </DataGrid.Columns>
</DataGrid>
```

## Design-Time Data

### Design DataContext

```xml
<Window xmlns:vm="using:MyApp.ViewModels"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyApp.Views.MainWindow">

    <Design.DataContext>
        <vm:MainViewModel />
    </Design.DataContext>

    <StackPanel Spacing="10">
        <TextBlock Text="{Binding Title}" FontSize="24" FontWeight="Bold" />
        <TextBlock Text="{Binding Description}" TextWrapping="Wrap" />
    </StackPanel>
</Window>
```

### Design Data in ViewModel

```csharp
using Avalonia.Styling;
using CommunityToolkit.Mvvm.ComponentModel;
using System.Collections.ObjectModel;

public partial class MainViewModel : ObservableObject
{
    [ObservableProperty] public partial string Title { get; set; } = string.Empty;
    [ObservableProperty] public partial ObservableCollection<User> Users { get; set; } = new();

    public MainViewModel()
    {
        if (Design.IsDesignMode)
        {
            Title = "Sample Title";
            Users = new ObservableCollection<User>
            {
                new() { Name = "John Doe", Email = "john@example.com" },
                new() { Name = "Jane Smith", Email = "jane@example.com" }
            };
        }
        else
        {
            LoadUsers();
        }
    }

    private void LoadUsers()
    {
        // Runtime: resolve IDataService (or IUserQuery façade) and populate Users
    }
}
```

## Common Patterns

### Master-Detail Pattern

```xml
<Grid ColumnDefinitions="200,*">
    <!-- Master list -->
    <ListBox Grid.Column="0"
             ItemsSource="{Binding Items}"
             SelectedItem="{Binding SelectedItem, Mode=TwoWay}" />

    <!-- Detail view -->
    <ContentControl Grid.Column="1"
                    Content="{Binding SelectedItem}">
        <ContentControl.ContentTemplate>
            <DataTemplate>
                <StackPanel Margin="10">
                    <TextBlock Text="{Binding Name}" FontSize="20" FontWeight="Bold" />
                    <TextBlock Text="{Binding Description}" TextWrapping="Wrap" Margin="0,10,0,0" />
                </StackPanel>
            </DataTemplate>
        </ContentControl.ContentTemplate>
    </ContentControl>
</Grid>
```

### Tab Navigation

```xml
<TabControl SelectedIndex="{Binding SelectedTabIndex, Mode=TwoWay}">
    <TabItem Header="Home">
        <views:HomeView DataContext="{Binding HomeViewModel}" />
    </TabItem>
    <TabItem Header="Settings">
        <views:SettingsView DataContext="{Binding SettingsViewModel}" />
    </TabItem>
    <TabItem Header="About">
        <views:AboutView DataContext="{Binding AboutViewModel}" />
    </TabItem>
</TabControl>
```

### Loading State

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class DataViewModel : ObservableObject
{
    [ObservableProperty] public partial bool IsLoading { get; set; }
    [ObservableProperty] public partial ObservableCollection<Item> Items { get; set; } = new();

    public DataViewModel(IDataService service) => _service = service;
    private readonly IDataService _service;

    [RelayCommand]
    private async Task LoadAsync(CancellationToken ct)
    {
        IsLoading = true;
        try
        {
            var data = await _service.FetchDataAsync(ct);
            Items = new ObservableCollection<Item>(data);
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

```xml
<Panel>
    <ListBox ItemsSource="{Binding Items}" />

    <!-- Loading overlay -->
    <Border Background="#80000000" IsVisible="{Binding IsLoading}">
        <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
            <ProgressBar IsIndeterminate="True" Width="200" />
            <TextBlock Text="Loading..." Foreground="White" Margin="0,10,0,0" />
        </StackPanel>
    </Border>
</Panel>
```

## Best practices

1. **Separation**: Views only present state; ViewModels coordinate; handlers/services own domain rules.
2. **Commands**: Prefer **`[RelayCommand]`** / `ICommand` bindings over code-behind event handlers.
3. **Validation**: Centralize with **FluentValidation**; surface errors as bindable messages (`messaging-logging-validation.md`).
4. **Async**: Use **`IAsyncRelayCommand`** or async `RelayCommand` methods with **`CancellationToken`**.
5. **Dispose**: Dispose subscriptions and long-lived services at app shutdown (Host disposal).
6. **Testing**: ViewModels and pure reducers/stores test without Avalonia head.
7. **Design-time data**: Use `Design.DataContext` and `Design.IsDesignMode` for preview.
8. **Memory**: Unsubscribe from long-lived events; avoid capturing `Window` in static lambdas.
9. **Change notification**: Trust **`ObservableObject`** code generation; avoid silent property sets.
10. **Single DI graph**: Resolve services from **Generic Host**, not ad hoc `new` for shared dependencies.

