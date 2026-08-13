# Chapter 2 — Dependency Properties

## 11. What is a Dependency Property?
A Dependency Property is a property managed by WPF's property system. It provides features such as styles, animation, data binding, inheritance, default values, metadata, validation, coercion, and property-value precedence.

## 12. Why does WPF need Dependency Properties instead of normal CLR properties?
Normal CLR properties do not provide the centralized metadata and value-resolution infrastructure WPF needs for binding, styles, animation, inheritance, and change notification. Dependency properties integrate these features into one property system.

## 13. CLR property vs Dependency Property
A CLR property stores/gets a value through normal .NET code. A dependency property is registered with WPF and normally wraps `GetValue`/`SetValue`. The CLR wrapper provides convenient syntax while the actual value is managed by the dependency-property system.

## 14. What is Property Value Precedence?
WPF can receive values from multiple sources: animations, local values, bindings, styles, templates, inheritance, defaults, and others. The effective value is selected according to a defined precedence order. A local value normally outranks a Style Setter, but an animation can override a local value.

## 15. What is DependencyObject?
`DependencyObject` is the base class that participates in WPF's dependency-property system. Objects deriving from it can store dependency-property values and participate in related WPF infrastructure.

## 16. Why register a Dependency Property?
Registration tells WPF the property's name, owner type, value type, metadata, and optional validation. WPF then knows how the property participates in its property system.

## 17. What is PropertyMetadata?
Metadata describes behavior associated with a dependency property, including its default value and callbacks. `FrameworkPropertyMetadata` adds framework-specific flags such as affecting measure, arrange, render, or inheritance.

## 18. What is a property changed callback?
It is invoked when the effective property value changes. It is commonly used to update dependent state or invalidate related behavior.

## 19. What is a coercion callback?
Coercion can constrain an effective value without necessarily rejecting the requested value. For example, a maximum value can be coerced down when it exceeds another property's current limit.

## 20. What is a validation callback?
Validation determines whether a proposed value is valid. A failed validation rejects the value rather than merely changing it to another acceptable value.

## 21. What is FrameworkPropertyMetadata?
It extends property metadata with flags controlling framework behavior, such as whether a property affects measure, arrange, render, inheritance, or data binding.

## 22. Register vs RegisterAttached vs AddOwner
`Register` creates a normal dependency property on an owner type. `RegisterAttached` creates an attached property intended to be set on other objects. `AddOwner` lets another type participate in an existing dependency property, optionally supplying new metadata.

## 23. What is an attached property?
An attached property is a dependency property whose value is stored on another dependency object. Examples include `Grid.Row` and `Canvas.Left`. It is useful when a parent or service needs to associate metadata with arbitrary child elements.

## 24. Can an attached property be used without being an attached Dependency Property?
A normal CLR property cannot provide the WPF attached-property behavior used by XAML and the dependency-property system. An attached property in WPF is normally registered through `RegisterAttached`.

## 25. How does Dependency Property inheritance work?
Some dependency properties are marked with inheritance metadata. A child can then obtain the effective value from an ancestor when it has no higher-precedence value of its own. `DataContext` is a major practical example.

### Scenario: local value, Style Setter, Trigger, animation, default
The animation value has higher precedence than a normal local value while the animation is active. A local value generally outranks Style Setters and ordinary Style triggers. The default value is among the lowest-precedence sources. For interview answers, emphasize that the exact precedence list matters and that triggers/templates have different positions depending on where they originate.
