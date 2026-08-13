# Chapter 8 — Threading and Dispatcher

## 93. Why can't most WPF controls be updated from a background thread?
WPF UI objects have thread affinity. They are generally associated with the thread on which they were created, normally the UI thread. Cross-thread access can cause exceptions or unsafe behavior.

## 94. What is the UI thread?
It is the thread running the WPF application's Dispatcher and normally responsible for input processing, layout, rendering coordination, and UI object interaction.

## 95. What is Dispatcher?
A `Dispatcher` queues work for execution on a specific thread. WPF uses the UI thread's Dispatcher to schedule UI work.

## 96. What is DispatcherObject?
`DispatcherObject` is a WPF base class for objects associated with a Dispatcher. `CheckAccess` and `VerifyAccess` can be used to reason about thread ownership.

## 97. What is thread affinity?
Thread affinity means an object is associated with a particular thread and should be accessed through that thread's Dispatcher when required.

## 98. How do you update UI from a background thread?
Use the UI Dispatcher, for example `Application.Current.Dispatcher.InvokeAsync(...)`, or structure the code with `async/await` so the UI continuation executes on the UI context.

## 99. Invoke vs BeginInvoke
`Invoke` executes synchronously and waits for the Dispatcher operation to finish. `BeginInvoke` queues work asynchronously and returns without waiting for completion.

## 100. What is InvokeAsync?
It schedules work on the Dispatcher and returns a task-like operation that can be awaited.

## 101. How does async/await interact with WPF?
When an async method awaits without opting out of context capture, the continuation normally resumes on the captured WPF synchronization context/UI thread.

## 102. What is SynchronizationContext?
It abstracts how work is posted back to a particular execution environment. WPF installs a synchronization context associated with its Dispatcher on the UI thread.

## 103. Why does awaited code usually return to the UI thread?
Because the await captures the current synchronization context when appropriate. After the awaited operation completes, the continuation is posted back through that context.

## 104. What happens with ConfigureAwait(false)?
The continuation does not require resuming on the captured synchronization context. This is useful in library code, but UI code must return to the UI thread before touching thread-affine UI objects.

### Scenario
`Task.Run` executes `DoWork` on a ThreadPool thread. The UI thread awaits it and is free to process messages. When the task completes, the continuation after `await` normally posts back to the WPF UI synchronization context, so `textBox.Text = result` runs on the UI thread.
