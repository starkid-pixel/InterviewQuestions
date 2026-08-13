# Chapter 17 — Advanced WPF

## 190. FrameworkElement vs Control
`FrameworkElement` provides layout, data binding support, styles/resources, and other framework-level behavior. `Control` adds control-specific features such as a template and properties related to foreground/background/font and user interaction semantics.

## 191. What is ContentControl?
A control designed to display one piece of content, optionally using a `DataTemplate` or control template.

## 192. What is ItemsControl?
A control that generates containers for a collection of items.

## 193. What is Control?
A base class for interactive controls that supports templating and common control properties.

## 194. Why does Control have a Template?
Templating separates a control's behavior/API from its visual representation, enabling restyling without rewriting the control's logic.

## 195. What is FrameworkElement responsible for?
Layout participation, alignment, sizing, data context, styles/resources, and other framework-level UI behavior.

## 196. What is Visual?
`Visual` is a core element of WPF's visual tree and rendering/composition system. It provides relationships between visual objects and transformation/coordinate-related infrastructure.

## 197. What is UIElement?
`UIElement` adds core input, hit testing, routed events, keyboard/mouse interaction, layout participation, and rendering invalidation behavior over the visual layer.

## 198. Explain WPF inheritance hierarchy.
A simplified path is `DependencyObject → Visual → UIElement → FrameworkElement → Control`. Not every WPF type follows this exact chain; for example, `FrameworkContentElement` is for content elements.

## 199. What are adorners?
Adorners are special visual elements used to decorate or overlay another element, often for selection handles, validation indicators, resize grips, or design-time UI.

## 200. What is AdornerLayer?
It is a layer that hosts adorners associated with elements in the relevant visual tree.

## 201. What is Freezable?
`Freezable` is a WPF base class for objects with change-notification behavior that can become immutable through `Freeze()`. Brushes, transforms, and animations contain common examples.

## 202. Why is Freezable special?
A frozen Freezable cannot change, enabling WPF to optimize it. Freezables also have special context/threading behavior within WPF's property system.

## 203. What does Freeze do?
It makes a Freezable immutable when it can be frozen. After freezing, attempting to modify it is invalid.

## 204. Why can freezing a Brush improve performance?
A frozen brush no longer needs change tracking for future modifications and can be shared efficiently. This can reduce overhead for reusable visual resources.

## 205. What is HwndSource?
It provides an HWND-backed presentation surface that can host WPF content and participate in interoperation with native Windows APIs.

## 206. How does WPF interoperate with Win32?
WPF can host or be hosted around native HWNDs using interop mechanisms such as `HwndHost` and `HwndSource`, with APIs for message/input integration.

## 207. What is WindowsFormsHost?
It hosts a Windows Forms control inside a WPF application.

## 208. What is HwndHost?
It is a base class used to host an HWND/native child window inside WPF.

## 209. What is airspace in WPF?
The airspace issue refers to restrictions caused by mixing WPF's retained/composited rendering with separate native HWND surfaces. Native child windows can appear above WPF content and do not behave like ordinary WPF visuals during transforms, clipping, and overlays.
