# UI — styling & theming (SukiUI)

> **Router**: `SKILL.md`. **~90 lines.** Keywords: `Selector`, `Style`, `SukiTheme`, `RequestedThemeVariant`, `ResourceDictionary`.

## Standard stack (this skill)

- **SukiUI**: **`SukiTheme`** + **`ThemeColor="Blue"`**; shell via **`SukiWindow`** where appropriate.
- **Dark**: **`Application`** **`RequestedThemeVariant="Dark"`**.
- **Icons**: **Material.Icons.Avalonia** → **`MaterialIconStyles`** in `Application.Styles`.
- Extra **`StyleInclude`** lines per **your pinned NuGet readme**.

## Selector cheatsheet

```xml
<Styles xmlns="https://github.com/avaloniaui">
    <Style Selector="Button">...</Style>
    <Style Selector="Button.Primary">...</Style>
    <Style Selector="Button:pointerover">...</Style>
    <Style Selector="TextBox:focus">...</Style>
</Styles>
```

## Resources

```xml
<Application.Resources>
    <SolidColorBrush x:Key="Accent" Color="#007ACC"/>
</Application.Resources>

<ResourceDictionary xmlns="https://github.com/avaloniaui">
    <ResourceDictionary.MergedDictionaries>
        <ResourceInclude Source="/Styles/Colors.axaml" />
    </ResourceDictionary.MergedDictionaries>
    <ResourceDictionary.ThemeDictionaries>
        <ResourceDictionary x:Key="Dark">
            <Color x:Key="Bg">#1E1E1E</Color>
        </ResourceDictionary>
    </ResourceDictionary.ThemeDictionaries>
</ResourceDictionary>
```

## Control templates (extract)

```xml
<Style Selector="Button.Primary">
    <Setter Property="Template">
        <ControlTemplate>
            <Border Background="{TemplateBinding Background}" CornerRadius="4" Padding="10,6">
                <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"
                                  Content="{TemplateBinding Content}"/>
            </Border>
        </ControlTemplate>
    </Setter>
</Style>
```

## DataTemplate

```xml
<DataTemplate x:Key="Row" DataType="{x:Type models:Person}">
    <StackPanel Orientation="Horizontal" Spacing="8">
        <TextBlock Text="{Binding Name}"/>
        <TextBlock Opacity="0.7" Text="{Binding Role}"/>
    </StackPanel>
</DataTemplate>
```

## Animations (AXAML)

```xml
<Style Selector="Button:pointerover">
    <Style.Animations>
        <Animation Duration="0:0:0.2" FillMode="Forward">
            <KeyFrame Cue="0%"><Setter Property="Opacity" Value="1"/></KeyFrame>
            <KeyFrame Cue="100%"><Setter Property="Opacity" Value="0.85"/></KeyFrame>
        </Animation>
    </Style.Animations>
</Style>
```

## Theme variant runtime

Toggle via `RequestedThemeVariant` or hosting APIs (**version-dependent Avalonia/SukiUI**).

---

Deeper catalogs: **[Avalonia docs](https://docs.avaloniaui.net)** + **SukiUI samples** — this skill keeps the file short for single-shot agent reads.
