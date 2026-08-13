# Chapter 3 — Data Binding

## 26. How does WPF data binding work internally?
A `Binding` describes how a target property obtains or sends a value. WPF creates a `BindingExpression` that connects the target dependency property to the source object/property, listens for relevant notifications, performs conversion, and updates one or both sides according to the binding mode.

## 27. Binding, BindingExpression, and binding source
`Binding` is the configuration. `BindingExpression` is the runtime object representing an active binding. The source is the object/property from which the value is obtained, while the target is normally a dependency property on a WPF element.

## 28. Source, ElementName, RelativeSource, and DataContext
`DataContext` supplies an inherited default source. `Source` explicitly specifies an object. `ElementName` identifies another named element. `RelativeSource` describes a relationship such as the current element, templated parent, or an ancestor.

## 29. What is DataContext?
`DataContext` is an inherited dependency property that commonly identifies the object used as the default binding source. It enables concise bindings such as `{Binding Name}`.

## 30. How does DataContext inheritance work?
A child normally inherits the parent's `DataContext` unless it explicitly sets another value. This follows the WPF property inheritance mechanism.

## 31. Why does DataContext sometimes disappear?
A child may explicitly set a different `DataContext`; a template or control may establish its own context; or the element may not be in the expected tree. `ContextMenu`, popups, and some generated template elements are common sources of confusion.

## 32. What happens when a binding cannot find its source property?
The binding generally produces a diagnostic message in the Output window and the target receives the binding's fallback/default behavior. WPF usually does not throw a normal exception for an ordinary missing property binding.

## 33. How do you debug binding failures?
Check the Visual Studio Output window for binding errors, verify `DataContext`, binding path, source type, spelling, and `DataContext` boundaries. Use `PresentationTraceSources.TraceLevel` for detailed binding diagnostics when necessary.

## 34. What is INotifyPropertyChanged?
It is an interface used by objects to notify consumers that a property value changed. WPF listens for `PropertyChanged` to update bindings when a source property changes.

## 35. Why is INotifyPropertyChanged necessary?
Without a notification, a binding may not know that a source property's value changed after the initial evaluation. The UI can therefore remain stale.

## 36. What happens if the ViewModel doesn't implement it?
The initial value can still appear, but later source-side changes normally won't automatically update the target.

## 37. What is UpdateSourceTrigger?
It controls when a target change is pushed back to the source. Common values are `PropertyChanged`, `LostFocus`, and `Explicit`. The default depends on the target dependency property.

## 38. OneWay, TwoWay, OneTime, OneWayToSource
`OneWay`: source → target. `TwoWay`: source ↔ target. `OneTime`: source → target once when the binding is established. `OneWayToSource`: target → source.

## 39. FallbackValue
It specifies a value to use when the binding cannot produce a value, such as when the source/path cannot be resolved.

## 40. TargetNullValue
It specifies what the target should receive when the binding successfully resolves but the source value is `null`.

## 41. What is a converter?
An `IValueConverter` transforms a value between source and target representations, such as converting a boolean to `Visibility`.

## 42. MultiBinding
`MultiBinding` combines several source values and passes them to an `IMultiValueConverter`, allowing a target value to depend on multiple inputs.

## 43. PriorityBinding
`PriorityBinding` lets WPF choose among bindings according to priority, using the best available result.

## 44. How does binding work with collections?
Items controls can bind their `ItemsSource` to an enumerable collection. WPF can create item containers and data templates for each item. Collection change notifications determine whether the UI reflects additions/removals automatically.

## 45. ObservableCollection vs List
`List<T>` does not notify WPF when items are added or removed. `ObservableCollection<T>` raises collection-change notifications, so controls can update their item containers incrementally. Neither automatically notifies when a property inside an existing item changes; the item itself needs property-change notification.

### Scenario: Customer.Name changes but UI stays stale
First verify that the bound `Customer` implements `INotifyPropertyChanged`, that the setter raises `PropertyChanged(nameof(Name))`, and that the ViewModel is modifying the same object used by the binding. Then verify `DataContext` and binding path. If the `Customer` reference itself changes, the containing property also needs notification.
