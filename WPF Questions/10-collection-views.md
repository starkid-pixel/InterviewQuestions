# Chapter 10 — Collection Views

## 116. What is ICollectionView?
It is a view over a collection that provides operations such as sorting, filtering, grouping, and current-item management without changing the underlying collection.

## 117. Collection vs collection view
The collection contains the data. The view provides a presentation-oriented projection over that data.

## 118. Sorting, filtering, grouping
Sorting changes presentation order, filtering determines which items are visible, and grouping organizes visible items into groups. These operations can be applied through `ICollectionView`.

## 119. What is CollectionViewSource?
It is a XAML-friendly way to create/configure a collection view, including sorting, grouping, filtering, and source configuration.

## 120. What is ICollectionView.Filter?
It is a predicate used to determine whether each item should be visible in the view.

## 121. Can multiple controls have different views over one collection?
Yes. Different controls can use different collection-view instances/configurations, allowing different sorting and filtering while sharing the same underlying data.

## 122. What is the default collection view?
WPF can provide a default view for a collection through collection-view infrastructure. Access through `CollectionViewSource.GetDefaultView(collection)` is a common way to obtain it.

## 123. What happens when two controls share the same collection?
If they use the same default view, view operations such as sorting/filtering/current item can affect both. Separate view instances avoid that shared presentation state.

## 124. How filter without modifying the collection?
Apply a filter to `ICollectionView` rather than removing items from the underlying collection. This preserves the source data and changes only its presentation.
