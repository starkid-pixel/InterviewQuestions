# Chapter 18 — XAML

## 210. How does XAML become objects?
WPF's XAML loader parses markup and constructs CLR objects, sets properties, creates collections, resolves resources/markup extensions, and connects names/events according to XAML rules.

## 211. What is x:Name?
It assigns a XAML name that can be used to identify an object in the generated namescope/code and by mechanisms such as `ElementName` binding.

## 212. Name vs x:Name
`Name` is itself a property on types that expose one. `x:Name` is a XAML directive that establishes a name in the XAML namescope. In many WPF elements they appear interchangeable, but `x:Name` works through XAML infrastructure rather than requiring a CLR `Name` property.

## 213. What is x:Type?
It produces a `System.Type` value for a specified XAML type, commonly used in styles, templates, and type-based resources.

## 214. What is x:Static?
It references a static field, property, or other static member exposed through XAML.

## 215. What is x:Reference?
It allows XAML to reference another named object, useful in scenarios where a direct binding or element reference is needed.

## 216. What are markup extensions?
They are XAML mechanisms that provide values dynamically or through special resolution logic. Examples include `{Binding}`, `{StaticResource}`, `{DynamicResource}`, and `{x:Static}`.

## 217. How does {Binding} work?
The binding markup creates a `Binding` object/configuration. WPF then creates a runtime binding expression on the target dependency property and resolves the source.

## 218. How does {StaticResource} work?
It requests a resource during XAML/resource resolution and assigns the resolved object to the target property.

## 219. How does {DynamicResource} work?
It establishes a deferred resource reference so WPF can resolve the resource at runtime and respond when the resource changes.

## 220. How create a custom markup extension?
Derive from `MarkupExtension` and implement `ProvideValue(IServiceProvider)`. Return the appropriate object/value or a deferred expression as required by the scenario.
