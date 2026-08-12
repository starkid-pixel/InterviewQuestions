# Task / Async-Await — Remaining Topics 1–40

## Table of Contents

1. [Task Exceptions](#1-task-exceptions)
2. [Cancellation](#2-cancellation)
3. [Task.Delay Vs Thread.Sleep](#3-task-delay-vs-thread-sleep)
4. [Task Completion](#4-task-completion)
5. [Task.Factory.Startnew](#5-task-factory-startnew)
6. [Task Scheduler](#6-task-scheduler)
7. [Threadpool](#7-threadpool)
8. [Deadlocks](#8-deadlocks)
9. [Synchronizationcontext](#9-synchronizationcontext)
10. [Parallel Vs Task](#10-parallel-vs-task)
11. [Valuetask](#11-valuetask)
12. [Semaphoreslim](#12-semaphoreslim)
13. [Lock / Monitor](#13-lock-monitor)
14. [Interlocked](#14-interlocked)
15. [Async Locking](#15-async-locking)
16. [Async Streams](#16-async-streams)
17. [Iasyncenumerable With Database/Api](#17-iasyncenumerable-with-database-api)
18. [Channels](#18-channels)
19. [Producer / Consumer](#19-producer-consumer)
20. [Concurrent Collections](#20-concurrent-collections)
21. [Taskcompletionsource](#21-taskcompletionsource)
22. [Async Event / Callback Patterns](#22-async-event-callback-patterns)
23. [Configureawait](#23-configureawait)
24. [Async Disposal](#24-async-disposal)
25. [Async File I/O](#25-async-file-i-o)
26. [Async Database Access](#26-async-database-access)
27. [Async Http](#27-async-http)
28. [Concurrency Vs Parallelism](#28-concurrency-vs-parallelism)
29. [Synchronous Vs Asynchronous](#29-synchronous-vs-asynchronous)
30. [Fire-And-Forget](#30-fire-and-forget)
31. [Background Services](#31-background-services)
32. [Async Job Queues](#32-async-job-queues)
33. [Throttling](#33-throttling)
34. [Parallel.Foreachasync](#34-parallel-foreachasync)
35. [Task.Yield](#35-task-yield)
36. [Async State Machine](#36-async-state-machine)
37. [Synchronous Completion](#37-synchronous-completion)
38. [Thread Affinity](#38-thread-affinity)
39. [Continuation Scheduling](#39-continuation-scheduling)
40. [Advanced Performance](#40-advanced-performance)

---

# 1. Task Exceptions

A Task can represent an operation that succeeds or fails.

Example:

```text
async Task DoWorkAsync()
{
    throw new Exception("Something went wrong");
}

```
The exception becomes associated with the Task.

If we await it:

```text
try
{
    await DoWorkAsync();
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}

```
The exception is observed when await resumes.

IMPORTANT:

```text
await Task
    |
    +---- operation succeeds
    |
    OR
    |
    +---- operation fails
               |
               v
           exception


```
With Wait():

```text
task.Wait();

```
exceptions can be exposed through AggregateException.

Example:

```text
try
{
    task.Wait();
}
catch (AggregateException ex)
{
    ...
}

```
This is one reason await is generally easier to use.


MULTIPLE TASKS

Example:

```text
Task t1 = Work1Async();
Task t2 = Work2Async();
Task t3 = Work3Async();

await Task.WhenAll(t1, t2, t3);

```
If multiple operations fail, WhenAll represents the combined
completion/failure of those operations.

IMPORTANT:

```text
Exceptions do not disappear just because the work is asynchronous.

```
They still need to be observed and handled.

---

# 2. Cancellation

Cancellation means:

```text
"Please stop this operation."

```
It is normally cooperative.

Example:

```text
CancellationTokenSource cts = new();

Task task = DoWorkAsync(cts.Token);

cts.Cancel();


```
The method can observe the token:

```text
async Task DoWorkAsync(CancellationToken token)
{
    while (true)
    {
        token.ThrowIfCancellationRequested();

        await Task.Delay(100, token);
    }
}


```
IMPORTANT:

CancellationToken does NOT forcibly kill a Thread.

It is a request.

The operation has to cooperate.


CancellationTokenSource

CancellationTokenSource:

```text
creates and controls the cancellation request.

```
CancellationToken:

```text
is passed to the operation.

```
Example:

```text
CancellationTokenSource cts = new();

await DoWorkAsync(cts.Token);

cts.Cancel();


```
Think:

```text
CancellationTokenSource
         |
         | Cancel()
         v
CancellationToken
         |
         v
   Running operation
         |
         v
   notices cancellation

```
---

# 3. Task.Delay Vs Thread.Sleep

This is one of the best examples for understanding async.

Thread.Sleep:

```text
Thread.Sleep(5000);

```
means:

```text
"Block this Thread for 5 seconds."

```
The Thread cannot do other work during that time.


Task.Delay

```text
await Task.Delay(5000);

```
means:

```text
"Asynchronously wait for 5 seconds."

```
The method can suspend while the Thread is available for other work.

Conceptually:

```text
Thread.Sleep:

    Thread
      |
      +---- Sleep
      |
      | BLOCKED
      |
      +---- 5 seconds
      |
      v
    Continue


Task.Delay:

    Method
      |
      +---- await Delay
      |
      v
    Suspended
      |
      | 5 seconds
      |
      v
    Resume


```
IMPORTANT:

Task.Delay creates/returns a Task representing the delay.

Initially:

```text
Task incomplete

```
After the delay:

```text
Task complete

```
---

# 4. Task Completion

A Task can be:

```text
incomplete

```
or:

```text
complete


```
You can check:

```text
task.IsCompleted


```
Other useful properties:

```text
task.IsCompletedSuccessfully
task.IsFaulted
task.IsCanceled
task.Status



```
Task.CompletedTask

Represents a Task that is already complete.

Example:

```text
return Task.CompletedTask;



```
Task.FromResult

Creates an already-completed Task containing a result.

Example:

```text
Task<int> task = Task.FromResult(100);


```
The Task already contains:

```text
Result = 100

```
Therefore:

```text
int result = await Task.FromResult(100);

```
does not need to suspend for an asynchronous operation.



IMPORTANT

This is directly related to our previous discussion:

```text
await does NOT automatically mean suspension.

```
If:

```text
Task.IsCompleted == true

```
then await may continue immediately.

If:

```text
Task.IsCompleted == false

```
then await can suspend the method until completion.

---

# 5. Task.Factory.Startnew

Older code may use:

```text
Task.Factory.StartNew(() =>
{
    DoWork();
});


```
For simple CPU-bound work, modern code usually uses:

```text
Task.Run(() =>
{
    DoWork();
});


```
Why?

Task.Run is a simpler API designed for the common case of scheduling
work to the ThreadPool.

Task.Factory.StartNew provides more control, including:

- TaskScheduler
- TaskCreationOptions
- TaskContinuationOptions
- Parent/child task behavior


For normal application code:

```text
Task.Run(...)

```
is usually easier.

Task.Factory.StartNew is still useful in advanced scenarios.

---

# 6. Task Scheduler

TaskScheduler determines how Tasks are scheduled for execution.

Simplified model:

```text
Task
  |
  v
TaskScheduler
  |
  v
ThreadPool
  |
  v
Worker Thread


```
The default TaskScheduler generally schedules work to the ThreadPool.

IMPORTANT:

TaskScheduler does not mean:

```text
"Create a Thread for every Task."

```
It decides how Task execution is scheduled.


There are also custom TaskSchedulers.

This is an advanced topic used when you need specialized scheduling
behavior.

---

# 7. Threadpool

ThreadPool is a collection of reusable worker Threads maintained by
.NET.

Instead of:

```text
create Thread
execute work
destroy Thread

```
repeatedly, .NET can reuse worker Threads.

Example:

```text
Task.Run(() => Work1());

Task.Run(() => Work2());

Task.Run(() => Work3());


```
The ThreadPool may execute them using a smaller number of Threads.

Example:

```text
Task 1 ---> Thread 5
Task 2 ---> Thread 6
Task 3 ---> Thread 5


```
The same Thread can execute multiple Tasks at different times.



THREADPOOL STARVATION

If many ThreadPool Threads are blocked:

```text
Thread 1 -> Wait()
Thread 2 -> Wait()
Thread 3 -> Wait()
...
Thread 100 -> Wait()

```
there may be no immediately available worker Threads for new work.

This is called ThreadPool starvation.

This is one reason blocking with:

```text
.Wait()
.Result
Thread.Sleep()

```
can be dangerous in server applications.

---

# 8. Deadlocks

A deadlock occurs when two or more pieces of work are waiting for
each other and nobody can proceed.

Async code can also participate in deadlocks.

A classic pattern is:

```text
SomeAsync().Result;


```
or:

```text
SomeAsync().Wait();


```
In certain environments, the async operation needs to resume on a
context that is currently blocked waiting for the operation.

Conceptually:

```text
Main/UI Thread
     |
     +---- calls async method
     |
     +---- waits using .Result
     |
     X BLOCKED

Async operation
     |
     +---- wants to resume on UI Thread
                 |
                 X UI Thread is blocked

```
Neither can continue.


This is one reason:

```text
await SomeAsync();

```
is preferred over:

```text
SomeAsync().Result;


```
IMPORTANT:

Deadlock behavior depends on the environment.

It is especially important in:

- UI applications
- Classic ASP.NET


ASP.NET Core does not have the same SynchronizationContext behavior.

---

# 9. Synchronizationcontext

SynchronizationContext is an abstraction that can control where
asynchronous continuations run.

For example, UI applications normally have a UI thread.

Suppose:

```text
await SomeOperationAsync();


```
After the operation completes, the continuation may need to return
to the UI context so that UI controls can safely be accessed.

Conceptually:

```text
UI Thread
   |
   v
await
   |
   v
asynchronous operation
   |
   v
continuation
   |
   v
UI Thread


```
Classic ASP.NET also had a SynchronizationContext.

ASP.NET Core generally does not have the same request
SynchronizationContext.


This topic becomes especially important when learning:

```text
ConfigureAwait(false)

```
---

# 10. Parallel Vs Task

These concepts are related but solve different problems.


Parallel

Parallel is mainly designed for parallel CPU work.

Example:

```text
Parallel.For(0, 1000, i =>
{
    Process(i);
});


```
Multiple iterations can execute concurrently on multiple CPU cores.


Task

Task represents an operation.

Example:

```text
Task task = Task.Run(() => Process());



```
Task.WhenAll

WhenAll is useful for asynchronous operations.

Example:

```text
await Task.WhenAll(
    CallApi1Async(),
    CallApi2Async(),
    CallApi3Async()
);


```
Think:

```text
Parallel
    -> CPU parallelism


Task.WhenAll
    -> asynchronous concurrency

```
---

# 11. Valuetask

Normally asynchronous methods return:

```text
Task<T>


```
Sometimes high-performance APIs use:

```text
ValueTask<T>


```
ValueTask can reduce allocations in situations where the operation
frequently completes synchronously.

Example:

```text
ValueTask<int> GetValueAsync()
{
    ...
}


```
However, ValueTask has additional usage rules and complexity.

Therefore:

```text
Use Task by default.

```
Use ValueTask when there is a demonstrated performance reason.

---

# 12. Semaphoreslim

SemaphoreSlim is useful when you want to limit the number of
operations running concurrently.

Example:

```text
SemaphoreSlim semaphore = new SemaphoreSlim(5);


```
This means:

```text
maximum 5 operations at the same time.


```
Example:

```text
await semaphore.WaitAsync();

try
{
    await CallApiAsync();
}
finally
{
    semaphore.Release();
}


```
Conceptually:

```text
100 API requests
      |
      v
SemaphoreSlim(5)
      |
      +---- Request 1
      +---- Request 2
      +---- Request 3
      +---- Request 4
      +---- Request 5
      |
      v
   remaining requests wait


```
This is called throttling/concurrency limiting.

---

# 13. Lock / Monitor

lock is used for thread synchronization.

Example:

```text
lock (_lockObject)
{
    counter++;
}


```
It means:

```text
only one Thread at a time can enter the critical section.


```
Equivalent lower-level concept:

```text
Monitor.Enter()
Monitor.Exit()


```
IMPORTANT:

You normally cannot do:

```text
lock (_lock)
{
    await SomethingAsync();
}


```
because the lock is thread-oriented while await can suspend and
resume on a different Thread.


For async mutual exclusion, SemaphoreSlim is often used instead.

---

# 14. Interlocked

Interlocked provides atomic operations.

Example:

```text
Interlocked.Increment(ref counter);


```
This is useful when multiple Threads need to update a simple numeric
value safely.

Without synchronization:

```text
counter++;


```
is not necessarily atomic.


Interlocked operations include:

```text
Increment()
Decrement()
Exchange()
CompareExchange()


```
Interlocked is often more lightweight than a lock for simple atomic
operations.

---

# 15. Async Locking

Traditional lock:

```text
lock (_lock)
{
    DoWork();
}


```
is designed around synchronous Thread locking.

For asynchronous code, use:

```text
SemaphoreSlim semaphore = new(1, 1);


```
Then:

```text
await semaphore.WaitAsync();

try
{
    await DoSomethingAsync();
}
finally
{
    semaphore.Release();
}


```
This allows the caller to asynchronously wait for access rather
than blocking a Thread.

---

# 16. Async Streams

Normal enumeration:

```text
IEnumerable<T>


```
Asynchronous enumeration:

```text
IAsyncEnumerable<T>


```
Example:

```text
await foreach (var item in GetItemsAsync())
{
    Process(item);
}


```
This is useful when items arrive asynchronously over time.

Instead of:

```text
get everything
   |
   v
return List<T>


```
you can:

```text
item 1
   |
   v
item 2
   |
   v
item 3
   |
   v
...


```
This is useful for large datasets and streaming APIs.

---

# 17. Iasyncenumerable With Database/Api

Suppose a database contains:

```text
10 million records


```
Returning:

```text
List<Customer>

```
may require loading a huge amount of data into memory.

An async stream can allow processing as records become available.

Conceptually:

```text
Database
   |
   v
Customer 1 ---> process
Customer 2 ---> process
Customer 3 ---> process
   ...
Customer 10M


```
Example:

```text
await foreach (var customer in GetCustomersAsync())
{
    Process(customer);
}


```
This is especially useful for large data streams.

---

# 18. Channels

System.Threading.Channels provides an asynchronous producer/consumer
mechanism.

Example:

```text
Channel<Job> channel = Channel.CreateUnbounded<Job>();

```
Producer:

```text
await channel.Writer.WriteAsync(job);


```
Consumer:

```text
Job job = await channel.Reader.ReadAsync();


```
Conceptually:

```text
Producer
   |
   v
+---------+
| Channel |
+---------+
   |
   v
Consumer


```
The producer creates work.

The consumer processes work.


Channels are very useful for building in-memory asynchronous queues.

---

# 19. Producer / Consumer

Producer:

```text
creates work.


```
Consumer:

```text
processes work.


```
Example:

```text
API request
    |
    v
Producer
    |
    v
Queue
    |
    v
Consumer
    |
    v
Process job


```
Multiple consumers can process jobs:

```text
Queue
  |
  +---- Worker 1
  +---- Worker 2
  +---- Worker 3
  +---- Worker 4


```
This is a common backend architecture.



BOUNDED QUEUE

A bounded queue has a maximum capacity.

Example:

```text
capacity = 100


```
If the queue is full, the producer may have to wait.

This creates backpressure.

Backpressure means:

```text
"Don't allow producers to generate work faster than consumers
 can process it."

```
---

# 20. Concurrent Collections

Normal collections such as:

```text
List<T>
Dictionary<TKey,TValue>


```
are not automatically safe for concurrent modification by multiple
Threads.

.NET provides:

```text
ConcurrentQueue<T>
ConcurrentDictionary<TKey,TValue>
ConcurrentBag<T>


```
Example:

```text
ConcurrentQueue<Job> queue =
    new ConcurrentQueue<Job>();


```
Multiple Threads can safely interact with the collection using its
supported concurrent operations.

---

# 21. Taskcompletionsource

TaskCompletionSource allows you to create a Task manually and
complete it later.

Example:

```text
TaskCompletionSource<int> tcs =
    new TaskCompletionSource<int>();


```
Someone can await:

```text
int result = await tcs.Task;


```
Later another piece of code can complete it:

```text
tcs.SetResult(100);


```
Conceptually:

```text
Caller
   |
   v
await tcs.Task
   |
   | waiting
   |
   | another component
   v
tcs.SetResult(100)
   |
   v
caller resumes


```
This is useful when converting callback/event-based APIs into
Task-based APIs.


Other methods:

```text
SetResult()
SetException()
SetCanceled()


```
There is also:

```text
RunContinuationsAsynchronously


```
which can be important when controlling how continuations execute.

---

# 22. Async Event / Callback Patterns

Event handlers are a special case.

Example:

```text
button.Click += async (sender, e) =>
{
    await DoSomethingAsync();
};


```
Event handlers may use:

```text
async void


```
because the event model expects a void-returning method.

For normal methods, prefer:

```text
Task
Task<T>


```
rather than:

```text
async void


```
Why?

Because callers cannot await an async void method and exception
handling becomes harder.

---

# 23. Configureawait

Normally:

```text
await SomeOperationAsync();


```
may capture the current context.

ConfigureAwait(false):

```text
await SomeOperationAsync()
    .ConfigureAwait(false);


```
means:

```text
"I don't need to resume on the captured context."


```
This is especially relevant for library code and environments with
a SynchronizationContext.

In ASP.NET Core, there is generally no SynchronizationContext like
classic ASP.NET had, so ConfigureAwait(false) has less significance
for the usual request flow.

---

# 24. Async Disposal

Some resources need asynchronous cleanup.

.NET provides:

```text
IAsyncDisposable


```
and:

```text
DisposeAsync()


```
Example:

```text
await using var resource =
    GetResourceAsync();


```
This allows cleanup to happen asynchronously.


For example, database/network resources may need asynchronous
operations during disposal.


Normal IDisposable:

```text
using


```
Async disposable:

```text
await using

```
---

# 25. Async File I/O

.NET provides asynchronous file APIs.

Examples:

```text
await File.ReadAllTextAsync(path);

await File.WriteAllTextAsync(path, data);


```
These are useful when file I/O may take significant time and you do
not want to block the calling Thread.


Do NOT automatically write:

```text
await Task.Run(() =>
    File.ReadAllText(path));


```
if a genuine async file API is available.


The normal pattern is:

```text
await File.ReadAllTextAsync(path);


```
If you then perform expensive CPU processing:

```text
string data = await File.ReadAllTextAsync(path);

var result = await Task.Run(() =>
    ExpensiveProcessing(data));


```
Here:

```text
File read
    -> async I/O


Processing
    -> CPU-bound

```
---

# 26. Async Database Access

Database libraries provide asynchronous methods such as:

```text
OpenAsync()
ExecuteReaderAsync()
ExecuteNonQueryAsync()
ExecuteScalarAsync()


```
Example:

```text
await connection.OpenAsync();

using var command = new SqlCommand(...);

using var reader =
    await command.ExecuteReaderAsync();


```
The database operation is I/O-bound.

Therefore:

```text
await ExecuteReaderAsync()


```
is generally preferred over:

```text
await Task.Run(() =>
    ExecuteReader());


```
Task.Run would just move the synchronous database call to another
ThreadPool Thread and make that Thread wait for the database.

---

# 27. Async Http

HttpClient provides asynchronous APIs.

Example:

```text
HttpClient client = new HttpClient();

HttpResponseMessage response =
    await client.GetAsync(url);


```
This is I/O-bound.

While the network operation is in progress, a Thread does not need
to remain blocked waiting for the response.


Therefore:

```text
await client.GetAsync(url);


```
is normally preferred over:

```text
await Task.Run(() =>
    client.GetAsync(url));


```
Do not use Task.Run unnecessarily around already asynchronous I/O.

---

# 28. Concurrency Vs Parallelism

These are related but different.


CONCURRENCY

Multiple operations are in progress during the same period.

Example:

```text
API Call 1
API Call 2
Database Call 3


```
They may all be waiting for external systems.


PARALLELISM

Multiple pieces of CPU work execute simultaneously on different
CPU cores.

Example:

```text
Core 1 -> Calculate A
Core 2 -> Calculate B
Core 3 -> Calculate C
Core 4 -> Calculate D


```
Think:

```text
Concurrency:
    "Multiple things are in progress."


Parallelism:
    "Multiple things are executing at the same time."


```
Async programming is often about concurrency.

Parallel programming is often about CPU parallelism.

---

# 29. Synchronous Vs Asynchronous

Synchronous:

```text
Do operation
   |
   v
wait here
   |
   v
operation completes
   |
   v
continue


```
Asynchronous:

```text
Start operation
   |
   v
await
   |
   v
method can suspend
   |
   v
other work can execute
   |
   v
operation completes
   |
   v
method resumes


```
IMPORTANT:

Asynchronous does not necessarily mean parallel.

An asynchronous operation may execute sequentially but without
blocking the Thread while waiting.

---

# 30. Fire-And-Forget

Fire-and-forget means:

```text
"Start this operation, but don't await its completion."


```
Example:

```text
_ = SendEmailAsync();


```
This can be dangerous.

Problems:

1. Exceptions may not be handled.

2. The application/request may finish before the operation finishes.

3. Cancellation may not be managed.

4. Resources owned by the request may be disposed before the
   operation finishes.

For example, in ASP.NET:

```text
_ = ProcessOrderAsync();


```
is not a reliable background job mechanism.


Better approaches include:

```text
BackgroundService
Queue
Durable job system

```
---

# 31. Background Services

ASP.NET Core provides:

```text
BackgroundService

```
and:

```text
IHostedService


```
These are useful for long-running background processing.

Example concept:

```text
API
 |
 v
Queue
 |
 v
BackgroundService
 |
 v
Process job


```
The BackgroundService can continuously read jobs and process them.


It also receives cancellation when the application shuts down.

---

# 32. Async Job Queues

Suppose an API receives:

```text
POST /generate-report


```
Generating the report takes 30 seconds.

You may not want the HTTP request to perform all 30 seconds of work.

Instead:

```text
API
 |
 v
Queue job
 |
 v
Return job ID
 |
 v
Background worker
 |
 v
Generate report


```
Task.Run alone is NOT a durable job queue.

For example:

```text
_ = Task.Run(() => GenerateReport());


```
If the application crashes, that work can disappear.

A real job queue provides more reliable processing.


Possible architecture:

```text
API
 |
 v
Queue
 |
 v
Worker
 |
 v
Database / storage

```
---

# 33. Throttling

Suppose you have:

```text
10,000 API calls


```
and you do:

```text
await Task.WhenAll(all10,000);


```
You may create too much concurrency.

The API server may reject requests.

The database may become overloaded.

Throttling means:

```text
"Limit how many operations can execute concurrently."


```
SemaphoreSlim example:

```text
SemaphoreSlim semaphore = new(10);


```
Only 10 operations can enter at once.

Conceptually:

```text
10,000 operations
      |
      v
SemaphoreSlim(10)
      |
      +---- 10 active
      |
      +---- remaining wait


```
This protects downstream systems.

---

# 34. Parallel.Foreachasync

Modern .NET provides:

```text
Parallel.ForEachAsync()


```
Example:

```text
await Parallel.ForEachAsync(
    items,
    async (item, token) =>
    {
        await ProcessAsync(item, token);
    });


```
It combines:

```text
iteration
+
asynchronous operations
+
concurrency limiting


```
You can configure:

```text
MaxDegreeOfParallelism


```
Example concept:

```text
MaxDegreeOfParallelism = 5


```
means approximately:

```text
maximum 5 operations concurrently.


```
This is different from:

```text
Parallel.ForEach()


```
which is designed primarily around synchronous work.

---

# 35. Task.Yield

Task.Yield() tells the async method to yield execution.

Example:

```text
await Task.Yield();


```
It can force the continuation to be scheduled asynchronously.

IMPORTANT:

Task.Yield() does NOT mean:

```text
"Run this on another Thread."


```
It is not equivalent to:

```text
Task.Run()


```
It is also not a timer.

Compare:

```text
Task.Yield()
    -> yield execution


Task.Delay(1000)
    -> asynchronously wait approximately 1 second


Task.Run(...)
    -> schedule synchronous work on ThreadPool

```
---

# 36. Async State Machine

When you write:

```text
async Task FooAsync()
{
    await SomethingAsync();

    DoSomethingElse();
}


```
the C# compiler transforms the async method into a state-machine
structure.

You don't normally write this state machine yourself.

Conceptually:

```text
State 1:
    Start FooAsync
    |
    v
    await SomethingAsync


If incomplete:

    save state
    return Task
    |
    v
    caller continues


Later:

    operation completes
    |
    v
    continuation resumes
    |
    v
    State 2:
    DoSomethingElse()


```
This is one reason async/await does not require a dedicated Thread.

---

# 37. Synchronous Completion

An async-looking method can sometimes complete synchronously.

Example:

```text
Task<int> GetValueAsync()
{
    return Task.FromResult(100);
}


```
Caller:

```text
int value = await GetValueAsync();


```
The Task is already complete.

Therefore await does not necessarily suspend.


This is extremely important.

The correct mental model is:

```text
await checks the Task


Task complete?
    |
    +---- YES --> continue
    |
    +---- NO ---> suspend and resume later


```
NOT:

```text
await always means:
    "go to another Thread"

```
---

# 38. Thread Affinity

Thread affinity means some operations must happen on a particular
Thread.

The classic example is a UI.

For example:

```text
UI controls

```
usually must be modified from the UI Thread.


An async operation might be:

```text
await DownloadDataAsync();


```
After completion, the continuation may need to return to the UI
Thread so that UI updates are safe.


Conceptually:

```text
UI Thread
   |
   v
await
   |
   v
Network operation
   |
   v
resume on UI context
   |
   v
update UI


```
Backend applications generally have fewer Thread-affinity
requirements.

---

# 39. Continuation Scheduling

After an awaitable operation completes, the code after await needs
to continue executing.

That continuation needs to be scheduled.

Depending on the environment and context, it may run through:

```text
SynchronizationContext

```
or:

```text
TaskScheduler / ThreadPool


```
Example:

```text
await GetDataAsync();

Console.WriteLine("Continue");


```
The code after await is the continuation.

Conceptually:

```text
GetDataAsync()
     |
     v
   await
     |
     v
Task incomplete
     |
     v
method suspended
     |
     |
     | operation completes
     v
continuation scheduled
     |
     v
Console.WriteLine(...)


```
ContinueWith also explicitly represents this concept:

```text
task.ContinueWith(...)


```
The continuation is associated with completion of the antecedent Task.

---

# 40. Advanced Performance

Once you understand all the fundamentals, performance becomes the
next level.


TASK ALLOCATIONS

Creating Tasks can involve allocations and bookkeeping.

For normal application code this is usually fine.

For extremely high-performance code, excessive allocations can
matter.


VALUETASK

ValueTask can sometimes reduce allocations when operations frequently
complete synchronously.

But it introduces additional complexity.

Therefore:

```text
Task by default.

ValueTask only when performance measurements justify it.



```
ASYNC STATE MACHINE OVERHEAD

async/await has compiler/runtime machinery.

For most application code this overhead is small compared with the
I/O operation itself.

Example:

```text
Database call takes 100 ms.

```
The async state-machine overhead is generally insignificant compared
with the database operation.

For extremely hot, tiny methods, however, performance can matter.



THREADPOOL STARVATION

If too many ThreadPool threads are blocked:

```text
.Wait()
.Result
Thread.Sleep()

```
the ThreadPool may struggle to service new work.

This can cause:

```text
increased latency
request delays
throughput reduction



```
EXCESSIVE TASK.RUN

Do not blindly write:

```text
await Task.Run(() => Everything());


```
Task.Run consumes a ThreadPool worker.

If the underlying operation is already asynchronous I/O, Task.Run
adds unnecessary ThreadPool usage.


Bad pattern:

```text
await Task.Run(() =>
    httpClient.GetAsync(url));


```
Better:

```text
await httpClient.GetAsync(url);


```
For CPU-bound synchronous work:

```text
await Task.Run(() =>
    ExpensiveCalculation());


```
can be appropriate.



EXCESSIVE CONCURRENCY

More concurrency is not always better.

Example:

```text
100,000 simultaneous database queries


```
could overwhelm:

```text
database
connection pool
CPU
memory
network


```
Therefore:

```text
bounded concurrency

```
is often better.


Example:

```text
SemaphoreSlim(10)


```
means:

```text
only 10 operations at a time.



```
FINAL MENTAL MODEL

After completing these 40 topics, keep this model:

THREAD
```text
=
actual execution resource


```
TASK
```text
=
representation of an operation


```
TASK.RUN
```text
=
schedule synchronous work on ThreadPool


```
ASYNC
```text
=
method can participate in asynchronous control flow


```
AWAIT
```text
=
asynchronously wait for an operation when necessary


```
WAIT
```text
=
synchronously block the current Thread


```
WHENALL
```text
=
asynchronously wait for multiple operations


```
WAITALL
```text
=
synchronously block until multiple operations finish


```
CONTINUEWITH
```text
=
continuation after Task completion


```
TASK.DELAY
```text
=
asynchronous delay


```
THREAD.SLEEP
```text
=
block Thread


```
CANCELLATION
```text
=
cooperative request to stop


```
SEMAPHORESLIM
```text
=
limit concurrent operations


```
CHANNEL
```text
=
asynchronous producer/consumer queue


```
BACKGROUND SERVICE
```text
=
long-running application-managed background processing


```
PARALLEL
```text
=
CPU parallelism


```
ASYNC I/O
```text
=
concurrency without blocking a Thread while waiting for I/O


```
CPU-BOUND
```text
=
work primarily limited by CPU


```
I/O-BOUND
```text
=
work primarily waiting for external systems



```
END OF TOPICS 1-40

---
