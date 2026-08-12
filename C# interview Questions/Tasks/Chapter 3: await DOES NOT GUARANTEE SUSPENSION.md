================================================================
CHAPTER: await DOES NOT GUARANTEE SUSPENSION
================================================================

This is one of the most important concepts to understand
correctly about async/await.

Many developers initially think:

    "Whenever execution reaches await, the method is suspended."

This is NOT correct.

The correct statement is:

    "await suspends the async method only when the Task being
     awaited is incomplete."

If the Task is already complete, execution continues immediately.


================================================================
1. THE BASIC CODE
================================================================

Consider:

    var result = await CalculateAsync();


Do not think:

    await
      |
      v
    automatically suspend
      |
      v
    return to caller


Instead, think:

    CalculateAsync()
          |
          v
    returns a Task
          |
          v
    await examines the Task
          |
          v
    Is the Task complete?
          |
       +--+--+
       |     |
      YES    NO
       |     |
       v     v
    Continue  Suspend
    immediately async method
                 |
                 v
            Task completes
                 |
                 v
            Resume method


================================================================
2. WHAT HAPPENS BEFORE await?
================================================================

Consider:

    var result = await CalculateAsync();


The first thing that happens is:

    CalculateAsync()


is called.

The method starts executing.

It eventually returns a:

    Task<int>


Then await receives that Task.

Conceptually:

    CalculateAsync()
          |
          v
    Task<int>
          |
          v
        await
          |
          v
    Check Task state


Therefore, await does NOT magically move the call to another
place.

The called method gets a chance to execute first.


================================================================
3. CASE 1 — TASK IS ALREADY COMPLETE
================================================================

Example:

    Task<int> CalculateAsync()
    {
        return Task.FromResult(100);
    }


Caller:

    async Task TestAsync()
    {
        var result = await CalculateAsync();

        Console.WriteLine(result);
    }


What happens?

Step 1:

    TestAsync()
        |
        v
    CalculateAsync()


Step 2:

    CalculateAsync() executes:

        return Task.FromResult(100);


Step 3:

    Task<int> is returned.

But:

    Task.FromResult(100)

creates an ALREADY COMPLETED Task.


So:

    Task
      |
      v
    Completed


Step 4:

    await checks the Task.

It sees:

    Task is already complete.


Therefore:

    await
      |
      v
    NO SUSPENSION
      |
      v
    Continue immediately


So this code:

    var result = await CalculateAsync();

does NOT necessarily mean:

    "The method is suspended here."


In this example, it continues immediately.


================================================================
4. CASE 2 — TASK IS INCOMPLETE
================================================================

Now consider:

    async Task<int> CalculateAsync()
    {
        await Task.Delay(5000);

        return 100;
    }


Caller:

    async Task TestAsync()
    {
        var result = await CalculateAsync();

        Console.WriteLine(result);
    }


Now the execution is different.


Step 1:

    TestAsync()
        |
        v
    CalculateAsync()


Step 2:

    CalculateAsync() starts.


Step 3:

    It reaches:

        await Task.Delay(5000);


The delay Task is incomplete.


Therefore:

    CalculateAsync()
        |
        v
    returns an INCOMPLETE Task
        |
        v
    TestAsync() receives that Task


Step 4:

    TestAsync() reaches:

        await CalculateAsync();


The Task is incomplete.


Therefore:

    TestAsync()
        |
        v
    SUSPENDS
        |
        v
    Control can return to its caller


Step 5:

    Five seconds later:

        Task.Delay(5000)
              |
              v
          completes
              |
              v
        CalculateAsync()
              |
              v
          resumes
              |
              v
          returns 100
              |
              v
        CalculateAsync Task completes
              |
              v
          TestAsync()
              |
              v
          resumes


================================================================
5. THE GOLDEN RULE
================================================================

MEMORIZE THIS:

    await does NOT guarantee suspension.


Instead:

    await + COMPLETED Task
        =
    continue immediately


    await + INCOMPLETE Task
        =
    suspend async method


This is the most important rule in this chapter.


================================================================
6. IT DEPENDS ON THE TASK, NOT SIMPLY ON THE WORD "await"
================================================================

Do not say:

    "await makes the method asynchronous."


That statement is too simplistic.


Do not say:

    "await always suspends."


That is incorrect.


Instead say:

    "await asynchronously waits for a Task. If the Task is
     incomplete, the async method can suspend. If the Task is
     already complete, execution continues immediately."


The important object is:

    TASK


The important question is:

    Is the Task complete?


================================================================
7. DOES IT DEPEND ON THE CALLED FUNCTION?
================================================================

You may say informally:

    "It depends on the called function."

But a more technically accurate statement is:

    "It depends on the Task returned by the called operation
     and whether that Task is complete when await checks it."


For example:

    var result = await SomeMethodAsync();


The important sequence is:

    SomeMethodAsync()
          |
          v
    returns Task
          |
          v
    await checks Task
          |
          v
    Complete or incomplete?


Therefore, we care about the returned Task.


================================================================
8. SAME METHOD — DIFFERENT TIMING
================================================================

It is also important to understand that you cannot always
look at a method name and say:

    "This method always suspends."


For example:

    async Task<int> GetValueAsync()
    {
        if (cacheAvailable)
        {
            return 100;
        }

        await Task.Delay(5000);

        return 100;
    }


Depending on the situation:

    CACHE AVAILABLE
        |
        v
    Result may be immediately available
        |
        v
    Task may complete immediately


Or:

    CACHE NOT AVAILABLE
        |
        v
    Delay occurs
        |
        v
    Task remains incomplete
        |
        v
    Caller may suspend


Therefore, the exact runtime state of the Task matters.


================================================================
9. Task.FromResult EXAMPLE
================================================================

Consider:

    Task<int> GetNumberAsync()
    {
        int result = 100;

        return Task.FromResult(result);
    }


Someone may look at this and say:

    "It returns Task, therefore it is asynchronous."


That is incorrect.


The calculation:

    int result = 100;


happens synchronously.


Then:

    Task.FromResult(result);


creates an already completed Task.


Therefore:

    Task.FromResult
        =
    Completed Task


It does NOT mean:

    "Run this operation asynchronously."


This is an extremely important distinction.


================================================================
10. YOUR CALCULATE1000000 EXAMPLE
================================================================

Consider your original example:

    async Task CalculateSumAsync()
    {
        var result = await Calculate1000000Async();
    }


And:

    Task<int> Calculate1000000Async()
    {
        int sum = 0;

        for (int i = 0; i < 1_000_000; i++)
        {
            sum += i;
        }

        return Task.FromResult(sum);
    }


You originally thought:

    "There is await, so when execution reaches await,
     the method should suspend."


But look carefully.


The call:

    Calculate1000000Async()


executes the loop first.


    Calculate1000000Async()
             |
             v
       Execute loop
             |
             v
       Calculate sum
             |
             v
       Task.FromResult(sum)
             |
             v
       COMPLETED Task
             |
             v
       await sees completed Task
             |
             v
       Continue immediately


There is no asynchronous suspension at that await.


The CPU work already happened synchronously.


================================================================
11. HOW Task.Run CHANGES THE EXAMPLE
================================================================

Now change it to:

    Task<int> Calculate1000000Async()
    {
        return Task.Run(() =>
        {
            int sum = 0;

            for (int i = 0; i < 1_000_000; i++)
            {
                sum += i;
            }

            return sum;
        });
    }


Now:

    Task.Run()
        |
        v
    Schedule CPU work
        |
        v
    ThreadPool
        |
        v
    Task returned
        |
        v
    Task may be incomplete
        |
        v
    await checks Task
        |
        v
    If incomplete:
        suspend


The important point is NOT:

    "await caused Task.Run."


The important point is:

    Task.Run created/scheduled the CPU operation and returned
    a Task representing that operation.


Then:

    await


waits for that Task.


================================================================
12. Task.Run DOES NOT GUARANTEE THAT await WILL SUSPEND
================================================================

This is a subtle but important point.


Consider:

    var result = await Task.Run(() => 10);


The work is extremely small.

It is possible that:

    Task.Run()
        |
        v
    ThreadPool executes work
        |
        v
    Work completes
        |
        v
    await checks Task
        |
        v
    Task already complete


In that situation:

    await


does not need to suspend.


Therefore, do not memorize:

    "Task.Run means await will suspend."


Instead:

    "Task.Run schedules work on the ThreadPool and returns
     a Task. The Task may be incomplete or may have already
     completed by the time await checks it."


Timing matters.


================================================================
13. await IS NOT A THREAD COMMAND
================================================================

Another misconception is to think:

    await
      =
    "Switch to another Thread."


That is not what await means.


await is primarily about:

    ASYNCHRONOUS CONTROL FLOW


It allows the method to say:

    "I need the result of this Task before I can continue."


If the result is not ready:

    Suspend my async method.


When the result becomes ready:

    Resume my async method.


It does not inherently say:

    "Create a Thread."


It does not inherently say:

    "Switch Threads."


It does not inherently say:

    "Run the operation somewhere else."


================================================================
14. await DOES NOT PERFORM THE OPERATION
================================================================

Consider:

    var customer = await GetCustomerAsync();


The operation is:

    "Fetch customer from database."


The Task represents:

    "The database fetch operation."


await means:

    "I need the result of this Task before continuing."


The database infrastructure performs the database operation.


The Task represents/tracks the operation.


await controls how your method waits for the result.


Therefore:

    Operation
        =
    What needs to happen


    Task
        =
    Representation/tracking of that operation


    await
        =
    Asynchronous waiting/control flow


================================================================
15. DATABASE EXAMPLE
================================================================

Consider:

    var reader =
        await command.ExecuteReaderAsync();


What happens conceptually?


    ExecuteReaderAsync()
            |
            v
    Database operation starts
            |
            v
    Task<SqlDataReader> returned
            |
            v
    Is Task complete?
          /       \
        YES       NO
         |         |
         v         v
     Continue    Suspend
     immediately  method
                    |
                    v
              Database continues
                    |
                    v
              Database completes
                    |
                    v
                Task completes
                    |
                    v
              Method resumes
                    |
                    v
                reader ready


Again:

    await


does not itself make the database operation asynchronous.


The database API:

    ExecuteReaderAsync()


provides the asynchronous operation.


The returned Task represents that operation.


await allows your method to asynchronously wait for it.


================================================================
16. COMPLETE VS INCOMPLETE TASK
================================================================

COMPLETE TASK:

    Operation has already finished.

Example:

    Task.FromResult(100)


State:

    Completed


When awaited:

    await task;


Result:

    Continue immediately.


INCOMPLETE TASK:

    Operation has not finished yet.

Example:

    Task.Delay(5000)


State:

    Waiting/pending


When awaited:

    await task;


Result:

    Async method can suspend.


Later:

    Task completes


Then:

    Async method resumes.


================================================================
17. A SIMPLE ANALOGY
================================================================

Imagine ordering food.


OPERATION:

    "Prepare my food."


TASK:

    Your order/tracking object.


You place the order:

    Order #123


The order represents:

    "My food preparation operation."


Now ask:

    "Is my order ready?"


If it is already ready:

    Continue immediately.


If it is not ready:

    You don't need to stand at the kitchen counter
    blocking the entire time.

You can leave and come back when it is ready.


This is similar to:

    await


The important thing is:

    await does not mean "always leave."


If your food is already ready:

    You take it immediately.


Similarly:

    If the Task is already complete,
    await continues immediately.


================================================================
18. THE EXACT INTERVIEW ANSWER
================================================================

Question:

    "Does await always suspend execution?"


Strong answer:

    "No. await does not guarantee suspension. The awaited
     expression first produces a Task. If that Task is already
     complete, execution continues synchronously without
     suspending the async method. If the Task is incomplete,
     the async method can suspend and its continuation will
     resume when the Task completes."


================================================================
19. ANOTHER INTERVIEW QUESTION
================================================================

Question:

    "Does every await cause a thread switch?"


Answer:

    "No. An await does not inherently cause a thread switch.
     If the Task is already complete, execution can continue
     synchronously. If the Task is incomplete, the method
     suspends and later resumes according to the applicable
     scheduling and synchronization context."


================================================================
20. ANOTHER INTERVIEW QUESTION
================================================================

Question:

    "What determines whether an async method suspends?"


Answer:

    "The completion state of the Task being awaited. If the
     awaited Task is incomplete, suspension can occur. If it
     is already complete, the method can continue immediately."


================================================================
21. THE MOST IMPORTANT FLOW TO MEMORIZE
================================================================

When you see:

    var result = await SomeMethodAsync();


Think:

    STEP 1
    ------
    Call SomeMethodAsync()


    STEP 2
    ------
    SomeMethodAsync() executes until it returns a Task.


    STEP 3
    ------
    await gets that Task.


    STEP 4
    ------
    Check whether the Task is complete.


    STEP 5A
    -------
    If COMPLETE:

        Continue immediately.


    STEP 5B
    -------
    If INCOMPLETE:

        Save async state
        Suspend async method
        Return control to caller


    STEP 6
    ------
    When Task completes:

        Resume async method
        Continue after await


================================================================
22. FINAL MENTAL MODEL
================================================================

Do NOT think:

    await
      |
      v
    suspend


Think:

    await
      |
      v
    Check Task
      |
      +-----------------------+
      |                       |
      v                       v
    COMPLETE               INCOMPLETE
      |                       |
      v                       v
    Continue                Suspend
    immediately             async method
                              |
                              v
                         Task completes
                              |
                              v
                            Resume


================================================================
23. ONE SENTENCE TO MEMORIZE
================================================================

    "await does not guarantee suspension; it depends on the
     completion state of the Task returned by the awaited
     operation. A completed Task allows execution to continue
     immediately, while an incomplete Task can cause the async
     method to suspend until the operation completes."


================================================================
24. FINAL CONNECTION TO THREAD, TASK, AND ASYNC/AWAIT
================================================================

THREAD:

    Executes code.


TASK:

    Represents/tracks an operation.


Task.Run:

    Schedules synchronous work on the ThreadPool.


async:

    Enables asynchronous method/state-machine behavior.


await:

    Asynchronously waits for a Task.


IMPORTANT:

    await != Thread


    await != Task.Run


    async != Thread


    Task != Thread


    Task.FromResult != asynchronous execution


    await != guaranteed suspension


The correct mental model is:

    OPERATION
        |
        v
      TASK
        |
        v
    await Task
        |
        v
    Is Task complete?
       / \
     YES  NO
      |    |
      |    v
      |  Suspend async method
      |    |
      |    v
      |  Task completes
      |    |
      |    v
      +-> Resume
