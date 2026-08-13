# Chapter 16 — Reusable Component Design

## UserControl vs CustomControl

### UserControl

Use when composing existing controls:

```text
SearchPanel
 ├── TextBox
 ├── Search Button
 └── Clear Button
```

### CustomControl

Use when the component needs a reusable behavior/API and replaceable templates.

## SearchBox custom control

### Dependency properties

```csharp
public class SearchBox : Control
{
    static SearchBox()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(SearchBox),
            new FrameworkPropertyMetadata(typeof(SearchBox)));
    }

    public static readonly DependencyProperty TextProperty =
        DependencyProperty.Register(
            nameof(Text),
            typeof(string),
            typeof(SearchBox),
            new FrameworkPropertyMetadata(
                "",
                FrameworkPropertyMetadataOptions.BindsTwoWayByDefault));

    public string Text
    {
        get => (string)GetValue(TextProperty);
        set => SetValue(TextProperty, value);
    }

    public static readonly DependencyProperty WatermarkProperty =
        DependencyProperty.Register(
            nameof(Watermark),
            typeof(string),
            typeof(SearchBox),
            new PropertyMetadata("Search..."));

    public string Watermark
    {
        get => (string)GetValue(WatermarkProperty);
        set => SetValue(WatermarkProperty, value);
    }

    public static readonly DependencyProperty
        SearchCommandProperty =
        DependencyProperty.Register(
            nameof(SearchCommand),
            typeof(ICommand),
            typeof(SearchBox));

    public ICommand? SearchCommand
    {
        get => (ICommand?)GetValue(SearchCommandProperty);
        set => SetValue(SearchCommandProperty, value);
    }
}
```

## Generic.xaml

```xml
<Style TargetType="{x:Type local:SearchBox}">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="{x:Type local:SearchBox}">
                <Grid>
                    <TextBox
                        Text="{TemplateBinding Text}"
                        Tag="{TemplateBinding Watermark}" />

                    <Button
                        Content="Search"
                        Command="{TemplateBinding SearchCommand}" />
                </Grid>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## Design principles

The public API belongs to the control class.

The visual structure belongs to the template.

This allows:
- themes
- alternate templates
- styling
- reuse
- testing of behavior independently from visuals
