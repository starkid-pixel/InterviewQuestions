# Chapter 4 — DataContext and Binding Context

## 46. Where does DataContext come from?
It is a dependency property. Developers commonly assign it in XAML, code-behind, a ViewModel-first framework, or through an application/navigation mechanism. Once set, it normally inherits down the element tree.

## 47. How does DataContext propagate?
Because `DataContext` is inheritable, descendants use the nearest inherited value unless they explicitly set another value.

## 48. Why does DataContext behave unexpectedly inside ContextMenu?
A `ContextMenu` is hosted through a `Popup`, so it is not a normal child of the control's visual tree. It therefore does not automatically inherit the placement target's `DataContext` in the same way as ordinary descendants.

## 49. How can a ContextMenu access the parent ViewModel?
Use an explicit binding source, commonly `PlacementTarget.DataContext`, for example through `RelativeSource Self` on the `ContextMenu`.

## 50. How do you use RelativeSource to access a parent ViewModel?
Use `RelativeSource FindAncestor` when the desired object is an ancestor in the relevant tree. For example, a control can bind to a Window's `DataContext` through an ancestor binding.

## 51. What happens when a child sets DataContext?
That child and its descendants normally inherit the new context instead of the parent's context.

## 52. How do you bind to the Window while a child has another DataContext?
Use an explicit source such as `ElementName=RootWindow` or `RelativeSource FindAncestor`, then bind through its `DataContext`.

## 53. How do you bind to a static property?
Use a static source mechanism such as `x:Static`, or expose the value through an appropriate binding/service/resource design. Be cautious with static mutable state because it complicates testing and lifetime management.
