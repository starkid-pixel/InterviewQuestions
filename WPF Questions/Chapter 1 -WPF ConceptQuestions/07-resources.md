# Chapter 7 — Resources

## 83. What is StaticResource?
`StaticResource` resolves a resource during XAML loading/resource lookup. It is generally efficient and appropriate when the resource is not expected to be replaced dynamically.

## 84. What is DynamicResource?
`DynamicResource` keeps a deferred resource reference so the value can be resolved/re-resolved at runtime when the resource changes.

## 85. StaticResource vs DynamicResource
Use `StaticResource` for stable resources and `DynamicResource` when runtime resource replacement matters, such as themes. Dynamic resources carry more runtime overhead.

## 86. When use DynamicResource?
Theme switching, runtime resource replacement, or resources that may not be available until later in the resource lookup lifecycle are common cases.

## 87. How does resource lookup work?
WPF searches relevant resource scopes, typically starting near the requesting element and moving outward through parent/application/resource scopes, with merged dictionaries participating according to their lookup rules.

## 88. What is resource inheritance?
Resource lookup follows the element/resource hierarchy. A child can typically find resources defined by an ancestor or application-level resource scope.

## 89. Application.Resources vs Window.Resources vs control resources
Application resources are broadly available to the application. Window resources apply to that Window's resource scope and descendants. Control/element resources are more local and normally take precedence over broader scopes.

## 90. What are merged dictionaries?
They allow resource dictionaries to compose resources from separate XAML files. They are useful for themes, styles, localization, and modular UI resources.

## 91. How should large WPF applications structure resources?
Separate themes, common styles, control styles, converters, brushes, and feature-specific resources into dictionaries. Merge them at appropriate scopes and avoid putting everything into one enormous application dictionary.

## 92. What if two dictionaries contain the same key?
Lookup uses the applicable dictionary ordering and scope rules. In merged dictionaries, later dictionaries can override earlier entries for the same key in the relevant lookup scenario.
