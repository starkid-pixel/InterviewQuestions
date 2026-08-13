# Chapter 1 — WPF Architecture & Internals

## 1. What makes WPF different from Windows Forms?
WPF is built around a retained-mode rendering system, XAML, dependency properties, routed events, data binding, templates, styles, and a powerful composition/layout engine. Windows Forms is primarily a wrapper around Win32 controls and follows a more immediate, control-centric model. WPF makes visual appearance and behavior much more customizable and separates UI structure from application logic more naturally.

## 2. Explain the WPF architecture from application code down to rendering.
A typical path is: application/ViewModel → XAML/controls → `FrameworkElement`/`UIElement`/`Visual` → layout and rendering infrastructure → WPF composition/rendering system → graphics subsystem. `PresentationFramework` provides high-level controls, styles, templates, and data binding; `PresentationCore` provides core UI/rendering types; `WindowsBase` provides foundational types such as `DependencyObject`, `DispatcherObject`, and dependency-property infrastructure.

## 3. What are the roles of PresentationFramework, PresentationCore, WindowsBase, and milcore?
`WindowsBase` contains foundational WPF infrastructure. `PresentationCore` contains core UI, input, media, and visual types. `PresentationFramework` contains higher-level controls, data binding, styles, templates, and application services. `milcore` is an internal/native rendering and composition layer responsible for much of WPF's graphics composition.

## 4. What is the WPF visual tree? How is it different from the logical tree?
The visual tree represents the actual rendered visual objects. The logical tree represents the logical content structure of an application. Templates can add many visual elements without changing the logical relationship. Resource lookup, event routing, inheritance, and traversal can therefore behave differently depending on which tree is involved.

## 5. Why does WPF have both a logical tree and a visual tree?
They solve different problems. The logical tree describes application content and ownership, while the visual tree describes the concrete rendering structure. A `Button` may logically contain text, while its template creates borders, presenters, and other visuals.

## 6. How does WPF render a control?
A control participates in layout through `Measure` and `Arrange`, then renders itself or its template. Its visual children form part of the visual tree. Changes that affect layout or appearance invalidate the appropriate parts of the system, and WPF composes the resulting visual content for display.

## 7. What is retained-mode graphics, and how does WPF use it?
In retained-mode graphics, application code describes what should exist rather than manually redrawing every pixel on every frame. WPF retains a scene/visual representation and determines what needs to be updated. This supports composition, animation, transformations, and declarative UI.

## 8. What happens internally when you change a property on a WPF control?
For a dependency property, WPF evaluates the property's effective value using its precedence system. It may run validation, coercion, and change callbacks, then invalidate layout or rendering if required. Bindings, triggers, animations, and inheritance can also participate in determining the effective value.

## 9. What are Measure and Arrange passes?
`Measure` asks children how much space they need under an available constraint. `Arrange` gives each child its final slot and establishes its position and size. Layout can repeat when changes invalidate measurement or arrangement.

## 10. Why can changing one property cause a large part of the UI to re-render?
A property may affect layout, inherited values, templates, or rendering. A layout-affecting change can invalidate a subtree or force ancestor/descendant layout work. A rendering-affecting change can invalidate visuals. The exact scope depends on the property metadata and the dependency relationships.

### Scenario: 10,000 elements and parent resize
Resizing the parent can invalidate layout. WPF performs measure/arrange work for affected elements, then renders the resulting visual tree. Performance depends on tree depth, virtualization, layout complexity, templates, bindings, and whether elements can be skipped or reused. The important optimization is to reduce unnecessary elements and work rather than trying to manually redraw controls.
