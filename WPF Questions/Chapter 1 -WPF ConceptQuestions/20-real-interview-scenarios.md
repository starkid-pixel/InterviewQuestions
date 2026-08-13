# Chapter 20 — Real Interview Scenarios

## 1. WPF application freezes for 10 seconds after Search
Determine whether the work is CPU-bound, synchronous I/O, layout/rendering, or lock contention. Capture a UI-thread stack during the freeze. Move CPU-heavy work off the UI thread and use asynchronous I/O for I/O-bound operations. Do not blindly wrap every operation in `Task.Run`.

## 2. UI displays stale data
Verify `INotifyPropertyChanged`, correct property names, the actual object being modified, `DataContext`, binding path, and whether a replacement object is notified. Check Output-window binding diagnostics.

## 3. DataGrid with 50,000 records is slow
Check virtualization first, then measure data loading, sorting/filtering, template complexity, converters, bindings, grouping, and memory usage. Prefer paging/server-side operations for data that does not need to be loaded or displayed simultaneously.

## 4. CanExecute is true but Button disabled
Check whether the command raised `CanExecuteChanged`, whether `CommandManager` is being relied upon correctly, whether the Button's `Command` is the expected instance, and whether another condition/style is disabling the Button.

## 5. Binding works on one screen but not another
Compare `DataContext`, binding source, visual/logical tree location, template boundaries, `ElementName`, `RelativeSource`, and resource scopes. Use binding diagnostics rather than guessing.

## 6. Memory grows every time a Window opens/closes
Take heap snapshots after forced/normal GC at repeated lifecycle points. Inspect retention paths from surviving Window/ViewModel instances. Look for static events, long-lived services, timers, event aggregators, global collections, and Dispatcher callbacks.

## 7. Background service updates ObservableCollection
Do not mutate a UI-bound collection from an arbitrary worker thread unless the design explicitly supports it. Marshal collection updates to the UI Dispatcher, or use a thread-safe synchronization/collection pattern appropriate to the application.

## 8. Two Views need different filtering/sorting
Use separate `ICollectionView` instances over the same underlying collection, or separate `CollectionViewSource` configurations. Do not mutate the source collection simply to implement view-specific filtering.

## 9. Millions of records
Do not attempt to create a UI element for every record. Use server-side filtering/paging, incremental loading, data virtualization, UI virtualization, and lightweight item templates. Design the data-access layer so the UI requests only what it needs.

## 10. ViewModel has 5,000 lines
Identify distinct responsibilities and extract child ViewModels, application/domain services, validation components, navigation/dialog abstractions, and feature-specific commands. Keep the resulting objects cohesive rather than splitting merely to reduce line count.

# Additional Deep-Dive Questions

## 232. What is the most important thing to profile first?
Profile the user-visible bottleneck: UI-thread CPU, blocking call, memory growth, layout/rendering, data retrieval, or excessive allocations. The correct optimization depends on the bottleneck.

## 233. Why is "just use Task.Run" not a complete performance strategy?
`Task.Run` moves CPU work to the ThreadPool. It does not make synchronous I/O asynchronous, does not solve excessive UI work, and can create ThreadPool pressure if used indiscriminately.

## 234. Why is virtualization not enough for very large datasets?
Virtualization limits UI objects, but the application may still load, sort, filter, and retain millions of records in memory. Data virtualization/paging addresses the data layer as well.

## 235. What makes a senior WPF answer different?
A senior answer explains trade-offs, lifecycle, threading, performance, testability, and failure modes. It should identify how to diagnose a problem rather than immediately prescribe one API.
