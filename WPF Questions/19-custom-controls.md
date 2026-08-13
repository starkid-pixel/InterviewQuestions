# Chapter 19 — Custom Controls

## 221. UserControl vs Custom Control
A `UserControl` is usually a reusable composition of existing controls with a fixed XAML structure. A custom control derives from a control base class and is typically template-driven, allowing consumers to completely replace its visual appearance.

## 222. When create a UserControl?
Use it when you want to package a known composition of existing controls and the visual structure is part of the component.

## 223. When create a Custom Control?
Use it when you are defining a reusable control with a public API and want consumers/themes to be able to replace the visual template.

## 224. Why template-based custom controls?
The behavior remains in the control class while the visual representation can vary by theme or consumer. This is one of WPF's strongest customization mechanisms.

## 225. Where does default template go?
Conventionally in `Themes/Generic.xaml` for a custom-control library.

## 226. What is Generic.xaml?
It is the conventional theme resource dictionary containing default styles/templates for custom controls in a WPF control library.

## 227. What is DefaultStyleKeyProperty?
A control can override its default style key so WPF knows which default style/template to locate in theme resources.

## 228. How does a custom control find its default template?
WPF uses theme style lookup and the control's default style key, typically locating the default style in `Generic.xaml` or an appropriate theme dictionary.

## 229. What is OnApplyTemplate?
It is called when the control's template is applied. A custom control can use it to find named template parts and attach behavior.

## 230. How access named template elements?
Use `GetTemplateChild("PartName")` or `Template.FindName(...)` in the appropriate context. Template parts should be treated as implementation details and guarded against missing parts.

## 231. How expose custom Dependency Properties?
Register them with `DependencyProperty.Register`, provide CLR wrappers, and add metadata/callbacks where appropriate. This allows styles, bindings, templates, animation, and other WPF features to work naturally.

### Scenario: reusable SearchBox
Define dependency properties for search text, watermark, and relevant state; expose an `ICommand`-style search operation; implement clear/search behavior in the control; provide a default template in `Generic.xaml`; and use template parts/buttons through `OnApplyTemplate`. Keep visual structure in the template and behavior/public API in the control.
