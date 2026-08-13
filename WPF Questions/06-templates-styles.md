# Chapter 6 — Templates, Styles, Triggers and Controls

## 68. Style vs ControlTemplate
A Style sets properties and can contain setters/triggers. A `ControlTemplate` defines the visual structure of a control. A style can assign a template through its `Template` setter.

## 69. ControlTemplate vs DataTemplate
`ControlTemplate` defines how a control itself is visually constructed. `DataTemplate` defines how a data object is presented, typically inside an `ItemsControl`, `ContentControl`, or similar host.

## 70. When use DataTemplate?
Use it when the data type should have a particular visual representation, such as displaying a `Customer` with name, email, and status.

## 71. When use ControlTemplate?
Use it when changing the visual structure/appearance of a control while retaining the control's behavior and API.

## 72. What is ItemsPanelTemplate?
It specifies the panel used to lay out generated item containers, such as a `StackPanel`, `WrapPanel`, or virtualizing panel.

## 73. What is ContentPresenter?
It is a framework element commonly used by templates to display content according to content-related properties and templates.

## 74. What is ContentControl?
A control designed to display a single piece of content. It is a foundation for many templated content scenarios.

## 75. What is ItemsControl?
It displays a collection of items by generating item containers and presenting each item through an item template and items panel.

## 76. ListBox vs ItemsControl
`ListBox` derives from `Selector` and provides selection behavior. `ItemsControl` does not provide selection.

## 77. What is TemplateBinding?
It is an optimized way for a template to bind an element property to a property of the templated control.

## 78. TemplateBinding vs normal Binding
`TemplateBinding` is specifically for template-to-templated-control property connections and is lightweight. A normal binding can support more complex source resolution, converters, modes, and paths.

## 79. What is RelativeSource TemplatedParent?
It explicitly identifies the control to which the current template belongs. It is useful when a template element needs a property from the templated control.

## 80. What are triggers?
Triggers change property values or invoke actions when conditions are met. They are commonly used for visual states such as hover, disabled, selected, or validation states.

## 81. Property, Data, and Event triggers
A property trigger reacts to a dependency property's value. A data trigger reacts to a binding result. An event trigger reacts to an event and can start actions such as storyboard behavior.

## 82. How does a Style Setter interact with a local value?
A local value normally has higher precedence than a Style Setter. Therefore setting a property directly on an element can prevent an ordinary Style Setter from taking effect.

### Scenario: Style Trigger cannot change Background
Check property-value precedence. A local `Background="Red"` has higher precedence than many Style-provided values. Remove the local value or move the state logic to a higher-precedence mechanism appropriate for the design.
