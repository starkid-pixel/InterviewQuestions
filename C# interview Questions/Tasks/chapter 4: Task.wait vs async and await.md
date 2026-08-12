============================================================
TASK.WAIT vs ASYNC/AWAIT vs TASK.RUN
============================================================

1. THE BASIC IDEA
-----------------

A Task represents an operation that may complete now or later.

For example:

Task<Customer> task = GetCustomerAsync();

The Task represents:

    "The operation of getting the customer."

The Task is NOT necessarily a thread.

A Task gives us a way to track the operation:

    - Is it completed?
    - Did it fail?
    - Was it cancelled?
    - What result did it produce?
    - When will it complete?


============================================================
2. THREE DIFFERENT CONCEPTS
============================================================

These three things have different responsibilities:

    Task
       |
       +--> Represents/tracks an operation

    Task.Run()
       |
       +--> Schedules synchronous work on the ThreadPool

    await
       |
       +--> Asynchronously waits for a Task


Example:

    await Task.Run(() => Calculate());


Read this as:

    1. Task.Run:
       Put Calculate() on the ThreadPool.

    2. Task:
       Represent the Calculate operation.

    3. await:
       Asynchronously wait for that operation.


============================================================
3. WHAT DOES await DO?
============================================================

Example:

    async Task<int> GetResultAsync()
    {
        int result = await GetDataAsync();

        return result;
    }


If GetDataAsync() returns an incomplete Task:

    GetDataAsync()
          |
          v
    Task returned
          |
          v
        await
          |
          v
    Task incomplete?
          |
          v
    Suspend this async method
          |
          v
    Return control to caller
          |
          v
    DO NOT BLOCK CURRENT THREAD
          |
          v
    Operation completes
          |
          v
    Resume the async method
          |
          v
    return result


IMPORTANT:

await does NOT guarantee suspension.

If the Task is already complete:

    await completedTask

can continue immediately.

Therefore:

    await
       =
    asynchronously wait for a Task

NOT:

    await
       =
    always suspend


============================================================
4. WHAT DOES Task.Wait() DO?
============================================================

Example:

    Task task = DoSomethingAsync();

    task.Wait();


This means:

    "Wait synchronously until the Task completes."

The current thread is BLOCKED.

Conceptually:

    Current Thread
          |
          v
       task.Wait()
          |
          v
    THREAD BLOCKED
          |
          v
    Task completes
          |
          v
    Thread continues


Therefore:

    await task
        =
    asynchronous waiting

    task.Wait()
        =
    synchronous blocking


============================================================
5. WHAT DOES Task.Result DO?
============================================================

Example:

    Task<Customer> task = GetCustomerAsync();

    Customer customer = task.Result;


.Result also synchronously blocks if the Task isn't complete.

So:

    task.Wait()

and:

    task.Result

are both forms of synchronous blocking.

Difference:

    Wait()
        -> waits for completion

    Result
        -> waits for completion and returns the result


Generally prefer:

    var customer = await GetCustomerAsync();

over:

    var customer = GetCustomerAsync().Result;


============================================================
6. WHY IS await PREFERRED OVER Wait()?
============================================================

Consider:

    button.Click += (sender, e) =>
    {
        Task.Run(() => Calculate()).Wait();
    };


The UI thread is blocked.

Flow:

    UI Thread
        |
        +--> Task.Run()
        |
        +--> Wait()
        |
        +--> UI THREAD BLOCKED
        |
        |         ThreadPool
        |              |
        |              +--> Calculate()
        |              |
        |              +--> complete
        |
        +--> UI thread continues


Now compare:

    button.Click += async (sender, e) =>
    {
        await Task.Run(() => Calculate());
    };


Flow:

    UI Thread
        |
        +--> Task.Run()
        |
        +--> await
        |
        +--> event handler suspends
        |
        +--> UI thread is FREE
        |
        |         ThreadPool
        |              |
        |              +--> Calculate()
        |              |
        |              +--> complete
        |
        +--> event handler resumes


The second version doesn't block the UI thread.


============================================================
7. WHY DOES async APPEAR WITH await?
============================================================

If you write:

    await Task.Run(() => Calculate());


the containing method generally needs to be marked:

    async


Example:

    async Task<int> CalculateAsync()
    {
        return await Task.Run(() => Calculate());
    }


Why?

Because the compiler transforms the async method into a state machine
that can suspend at an incomplete await and resume later.


IMPORTANT:

async does NOT:

    - create a thread
    - automatically use the ThreadPool
    - make synchronous code asynchronous


async mainly allows the method to participate in asynchronous control
flow and use await.


============================================================
8. WHAT DOES Task.Run() DO?
============================================================

Example:

    Task.Run(() => Calculate());


Task.Run schedules the synchronous work for execution on the
ThreadPool.

Example:

    int Calculate()
    {
        // CPU-intensive calculation

        int sum = 0;

        for (int i = 0; i < 1_000_000_000; i++)
        {
            sum += i;
        }

        return sum;
    }


Then:

    await Task.Run(() => Calculate());


means:

    Task.Run
        |
        v
    ThreadPool
        |
        v
    Calculate()
        |
        v
    Task<int>
        |
        v
    await


Task.Run is useful when you have synchronous CPU-bound work and
you specifically want to execute it on a ThreadPool thread.


============================================================
9. WHEN SHOULD I USE async/await?
============================================================

Use async/await when you have an asynchronous operation.

Typical examples:

    - Database I/O
    - HTTP calls
    - Network I/O
    - File I/O
    - Stream I/O
    - Other APIs that provide genuine async operations


Example:

    async Task<Customer> GetCustomerAsync()
    {
        return await database.GetCustomerAsync();
    }


The database operation may take 2 seconds.

The important point is:

    The database is working,
    but a server thread does not have to sit there
    blocked for those 2 seconds.


Flow:

    Request Thread
         |
         v
    Start database I/O
         |
         v
       await
         |
         v
    Thread becomes available
         |
         |
         |     Database
         |         |
         |         v
         |    operation runs
         |         |
         |         v
         |     completes
         |
         v
    Task completes
         |
         v
    Method resumes


============================================================
10. WHEN SHOULD I USE Task.Run?
============================================================

Task.Run is primarily useful for synchronous CPU-bound work.

Example:

    int Calculate()
    {
        // CPU-intensive synchronous work
    }


Then:

    async Task<int> CalculateAsync()
    {
        return await Task.Run(() => Calculate());
    }


Conceptually:

    Current Thread
          |
          v
       Task.Run
          |
          v
      ThreadPool
          |
          v
      Calculate()
          |
          v
       CPU work


Task.Run is NOT normally used simply to make I/O asynchronous.


============================================================
11. DO NOT DO THIS FOR NATIVE ASYNC I/O
============================================================

Suppose the database already provides:

    GetCustomerAsync()


Do NOT normally do:

    await Task.Run(() =>
        database.GetCustomerAsync());


Why?

Because the database API is already asynchronous.

Prefer:

    await database.GetCustomerAsync();


Similarly, if an HTTP client provides:

    await httpClient.GetAsync(url);


don't normally do:

    await Task.Run(() =>
        httpClient.GetAsync(url));


Task.Run does not make an already-asynchronous I/O operation
"more asynchronous."


============================================================
12. BAD BACKEND PATTERN
============================================================

Avoid using Task.Run just to make synchronous database I/O
appear asynchronous:

    async Task<Customer> GetCustomerAsync()
    {
        return await Task.Run(() =>
            database.GetCustomer());
    }


What actually happens?

    Request Thread
         |
         v
      Task.Run
         |
         v
      ThreadPool
         |
         v
    database.GetCustomer()
         |
         v
    ThreadPool thread BLOCKED
         |
         v
    Database response
         |
         v
    ThreadPool thread continues


You have moved the blocking from one thread to another.

You have not achieved true asynchronous database I/O.


If the provider supports:

    database.GetCustomerAsync()


use:

    async Task<Customer> GetCustomerAsync()
    {
        return await database.GetCustomerAsync();
    }


============================================================
13. WHEN SHOULD I CALL Task.Wait()?
============================================================

General rule:

    Avoid Task.Wait() in asynchronous application code.


Task.Wait() is useful only when you deliberately want synchronous
blocking.

For example, in a truly synchronous context where you cannot
asynchronously propagate the operation, you may encounter:

    task.Wait();


But understand what you are choosing:

    Current thread
         |
         v
    BLOCKED
         |
         v
    Wait for Task
         |
         v
    Continue


You are giving up the non-blocking advantage of async/await.


============================================================
14. WHY NOT JUST USE Task.Wait() EVERYWHERE?
============================================================

Because it blocks threads.

Suppose a backend has many requests:

    Request 1 -> database -> thread blocked
    Request 2 -> database -> thread blocked
    Request 3 -> database -> thread blocked
    Request 4 -> database -> thread blocked
    ...
    Request 1000 -> database -> thread blocked


This can consume ThreadPool threads unnecessarily.

With proper async I/O:

    Request 1 -> database -> await -> thread available
    Request 2 -> database -> await -> thread available
    Request 3 -> database -> await -> thread available
    ...
    Request 1000 -> database -> await -> thread available


The database still takes the same amount of time.

Async does NOT magically make the database faster.

It allows the server to use its threads more efficiently while
waiting for I/O.


============================================================
15. async ALL THE WAY
============================================================

Suppose:

    Controller
       |
       v
    Service
       |
       v
    Repository
       |
       v
    Database


Database:

    Task<Customer> GetCustomerAsync()


Repository:

    async Task<Customer> GetCustomerAsync()
    {
        return await database.GetCustomerAsync();
    }


Service:

    async Task<Customer> GetCustomerAsync()
    {
        return await repository.GetCustomerAsync();
    }


Controller:

    async Task<IActionResult> GetCustomer()
    {
        var customer = await service.GetCustomerAsync();

        return Ok(customer);
    }


The Task propagates upward.


    Database
       |
       | Task<Customer>
       v
    Repository
       |
       | Task<Customer>
       v
    Service
       |
       | Task<Customer>
       v
    Controller


This is what people mean by:

    "async all the way."


It does NOT mean:

    "Every method in the application must be async."

It means:

    "Don't unnecessarily convert asynchronous waiting into
     synchronous blocking."


============================================================
16. IMPORTANT: NOT EVERY METHOD NEEDS async
============================================================

This method does not need async:

    Customer MapCustomer(CustomerEntity entity)
    {
        return new Customer
        {
            Id = entity.Id,
            Name = entity.Name
        };
    }


There is no asynchronous operation here.

Similarly:

    int Add(int a, int b)
    {
        return a + b;
    }


does not need:

    async Task<int> AddAsync(...)


Use async where there is a genuine reason to participate in
asynchronous control flow.


============================================================
17. A METHOD CAN RETURN A TASK WITHOUT USING async
============================================================

For example:

    Task<Customer> GetCustomerAsync(int id)
    {
        return repository.GetCustomerAsync(id);
    }


This is valid.

You don't need:

    async Task<Customer>

if you are simply passing the Task through.


Use:

    async Task<Customer> GetCustomerAsync(int id)
    {
        var customer = await repository.GetCustomerAsync(id);

        // Do something with customer

        return customer;
    }


when you actually need to await and continue processing.


============================================================
18. EVENT HANDLER EXAMPLE
============================================================

A UI event handler can be:

    button.Click += async (sender, e) =>
    {
        await Task.Run(() =>
        {
            Calculate();
        });
    };


Why?

    Event handler
         |
         v
    Task.Run
         |
         v
    CPU work on ThreadPool
         |
         v
    Task
         |
         v
    await
         |
         v
    event handler suspends
         |
         v
    UI thread remains available
         |
         v
    Task completes
         |
         v
    event handler resumes


The event handler is an important async boundary.


============================================================
19. TASK.RUN + await + Task.WAIT
============================================================

Compare these three:

A)

    await Task.Run(() => Calculate());


Meaning:

    Run Calculate on ThreadPool
    +
    asynchronously wait for completion


B)

    Task.Run(() => Calculate()).Wait();


Meaning:

    Run Calculate on ThreadPool
    +
    synchronously block current thread until completion


C)

    Calculate();


Meaning:

    Run Calculate synchronously on the current thread


Therefore:

    Calculate()
        |
        +--> current thread does CPU work


    Task.Run(() => Calculate())
        |
        +--> ThreadPool does CPU work


    await Task.Run(() => Calculate())
        |
        +--> ThreadPool does CPU work
        +--> current thread does not block while waiting


    Task.Run(() => Calculate()).Wait()
        |
        +--> ThreadPool does CPU work
        +--> current thread is blocked while waiting


============================================================
20. THE MOST IMPORTANT MENTAL MODEL
============================================================

Remember these four statements:

    TASK
    ----
    Represents/tracks an operation.


    Task.Run
    --------
    Schedules synchronous work on the ThreadPool.


    await
    -----
    Asynchronously waits for a Task.


    Task.Wait()
    ------------
    Synchronously waits and blocks the current thread.


============================================================
21. DECISION TABLE
============================================================

Situation:

    Database has ExecuteAsync()
        |
        +--> Use async/await


    HTTP client has GetAsync()
        |
        +--> Use async/await


    File API has ReadAsync()/WriteAsync()
        |
        +--> Use async/await


    CPU-intensive synchronous calculation
        |
        +--> Task.Run may be appropriate when you need
             to move the work off the current thread


    Need to synchronously block until a Task completes
        |
        +--> Task.Wait() / Result
             BUT understand that this blocks the thread
             and should generally be avoided in async code.


============================================================
22. INTERVIEW ANSWER
============================================================

Question:

"When do you use Task.Run versus async/await?"


Answer:

"Task.Run is primarily used to schedule synchronous CPU-bound work
on the ThreadPool. async/await is used to asynchronously wait for
an asynchronous operation without blocking the current thread.

For I/O-bound operations such as database, HTTP, or file I/O, if
the API provides a native asynchronous method, I use that method
with await rather than wrapping it in Task.Run.

I generally avoid Task.Wait() or Task.Result in asynchronous
application code because they synchronously block the current
thread and can reduce scalability or cause deadlocks in some
environments."


============================================================
23. ONE SENTENCE TO MEMORIZE
============================================================

    Task.Run = where the synchronous work executes.

    Task = represents the operation.

    await = how I asynchronously wait for the operation.

    Wait()/Result = synchronously block while waiting.


============================================================
24. FINAL EXAMPLE
============================================================

CPU-bound work:

    int Calculate()
    {
        // expensive CPU work
    }


Async wrapper:

    async Task<int> CalculateAsync()
    {
        return await Task.Run(() => Calculate());
    }


Caller:

    async Task ProcessAsync()
    {
        int result = await CalculateAsync();
    }


Higher caller:

    async Task HandleAsync()
    {
        await ProcessAsync();
    }


The flow is:

    HandleAsync
         |
         | await
         v
    ProcessAsync
         |
         | await
         v
    CalculateAsync
         |
         | await
         v
    Task.Run
         |
         v
    ThreadPool
         |
         v
    Calculate()


This is the "async propagation" concept.


============================================================
25. THE KEY CONCEPT YOU DISCOVERED
============================================================

If a method returns:

    Task<T>


the caller can do:

    T value = await MethodAsync();


If the caller uses await, the caller generally needs to participate
in asynchronous execution:

    async Task CallerAsync()
    {
        T value = await MethodAsync();
    }


That Task can then propagate upward:

    Caller A
       |
       | await
       v
    Caller B
       |
       | await
       v
    Caller C
       |
       | await
       v
    Async operation


This is why async naturally propagates upward through the call
chain.

You can stop the propagation with:

    .Wait()
    .Result

but that changes asynchronous waiting into synchronous blocking.


============================================================
END
============================================================
