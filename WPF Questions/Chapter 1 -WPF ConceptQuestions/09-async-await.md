# Chapter 9 — Async/Await in WPF

## 105. Why avoid Result/Wait on UI thread?
They block the UI thread. If the awaited operation needs to resume on that same UI context, the continuation cannot run because the UI thread is blocked, producing a classic deadlock scenario.

## 106. How does blocking cause deadlocks?
An async method may capture the UI synchronization context. The caller blocks the UI thread waiting for completion, while the async continuation waits for the UI thread. Neither can proceed.

## 107. Why can GetDataAsync().Result deadlock?
If `GetDataAsync` awaits something and captures the WPF context, its continuation requires the UI thread. `.Result` blocks that thread, preventing the continuation from running.

## 108. CPU-bound vs I/O-bound
CPU-bound work consumes processor time and can benefit from moving computation off the UI thread. I/O-bound work spends time waiting for external resources and should generally use asynchronous I/O APIs rather than occupying a ThreadPool thread with blocking waits.

## 109. Should Task.Run be used for database/network operations?
Normally use true asynchronous database/network APIs. `Task.Run` does not make a synchronous I/O API genuinely asynchronous; it merely moves the blocking work to another thread.

## 110. When use Task.Run in WPF?
It is useful for substantial CPU-bound work that would otherwise block the UI thread, provided the work is thread-safe and its results are marshalled back appropriately.

## 111. How cancel a long-running operation?
Use `CancellationToken`, pass it through the operation, and make the operation observe cancellation. Cancellation is cooperative; the code doing the work must honor the token.

## 112. How report progress?
Use a progress abstraction such as `IProgress<T>`/`Progress<T>`, or publish progress through a ViewModel property/observable mechanism. Ensure UI updates occur on the UI context.

## 113. How prevent multiple concurrent button operations?
Track an executing state and expose it through `CanExecute`, disable the command while running, or use an async-command implementation that prevents re-entry.

## 114. How handle exceptions from async void?
Only event handlers should normally be `async void`. Exceptions cannot be awaited by the caller, so event handlers need explicit exception handling. Prefer `Task`-returning methods elsewhere.

## 115. Why avoid async void in ViewModels?
`Task`-returning methods can be awaited, composed, tested, and have exceptions observed. Async commands are a better abstraction for asynchronous ViewModel operations.
