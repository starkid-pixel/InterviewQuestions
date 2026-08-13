# Chapter 11 — ItemsControl, ListBox, DataGrid and Virtualization

## 125. How does ItemsControl generate UI?
It uses an item container generator to create containers for source items and applies templates/styles to present each item.

## 126. What is an item container?
It is the UI element that hosts an individual data item. For example, a `ListBoxItem` is the container for an item in a `ListBox`.

## 127. Item vs container
The item is the data object. The container is the WPF element used to display and interact with that object.

## 128. What is virtualization?
Virtualization creates UI containers only for items that need to be displayed rather than creating a visual element for every item in a large collection.

## 129. How does VirtualizingStackPanel improve performance?
It limits the number of realized item containers, reducing visual-tree size, layout work, memory usage, and rendering cost.

## 130. Why can virtualization stop working?
Common causes include unsupported panel configurations, wrapping a virtualized control in an outer `ScrollViewer`, disabling virtualization, grouping/configuration choices, custom panels, or operations that require realizing many items.

## 131. UI virtualization vs data virtualization
UI virtualization limits generated UI elements. Data virtualization limits how much data is loaded/fetched in the first place. Large applications often need both.

## 132. How display 100,000 records efficiently?
Use UI virtualization, paging or data virtualization, server-side filtering/sorting where possible, lightweight templates, efficient bindings, and incremental loading. Avoid materializing and rendering unnecessary data.

## 133. Why can nested ScrollViewer break virtualization?
A virtualizing panel needs to know a meaningful viewport. An outer scroll container can provide an effectively unbounded measurement, causing the inner panel to realize far more items than intended.

## 134. Grouping and virtualization
Grouping can change the panel/tree structure and may disable or reduce virtualization depending on configuration. Verify the actual generated visual tree and virtualization settings rather than assuming it remains enabled.

### Scenario: 500,000-record DataGrid
First profile rather than guess. Check row/column virtualization, template complexity, converters, bindings, grouping, sorting, data loading, and memory usage. Prefer server-side filtering/paging and only materialize the records needed for the current view. Use a profiler and WPF diagnostic tooling to identify the actual bottleneck.
