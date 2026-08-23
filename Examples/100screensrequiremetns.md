# WPF Large Application Architecture

## 1. Problem Statement

Assume we are designing a WPF application with the following characteristics:

* UI-rich application
* Potentially 100+ screens
* Multiple business capabilities
* Expected to grow over time
* Need for maintainability and scalability

The important architectural principle is:

> **The number of screens alone does not determine the architecture.**

We should not immediately say:

> "The application has 100 screens, therefore we must use a Shell and ViewModel-first navigation."

Instead, we should evaluate the architecture through separate decisions.

---

# 2. The Three Architectural Decisions

We can separate the problem into three independent dimensions.

```text
                    Large WPF Application
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
         ▼                  ▼                  ▼

 Application            Navigation          Decomposition
 Composition              Target
```

## Question 1: How is the application composed?

Possible options:

```text
1. Shell
2. Multiple Windows
3. Page / Frame
```

## Question 2: What is the navigation target?

Possible options:

```text
1. View-first
2. ViewModel-first
```

The exact meaning depends on the application composition model.

## Question 3: How is the application organized?

Possible options:

```text
1. Technical-layer organization
2. Feature-based decomposition
```

For our large application, we consider **feature-based decomposition** a strong choice.

---

# 3. Important Principle: These Decisions Are Independent

We should not assume:

```text
Shell
    =
ViewModel-first
    =
Feature-based
```

These solve different problems.

For example:

```text
Shell
+ View-first
+ Feature-based
```

is possible.

Also:

```text
Shell
+ ViewModel-first
+ Feature-based
```

is possible.

Similarly:

```text
Multiple Windows
+ View-first
+ Feature-based
```

and:

```text
Multiple Windows
+ ViewModel-first
+ Feature-based
```

are also possible.

The architecture should be selected based on requirements.

---

# 4. Application Composition

## Option 1 — Shell

A Shell is appropriate when the application is conceptually one primary workspace.

For example:

```text
┌─────────────────────────────────────────────┐
│ Header / Toolbar / User Information         │
├───────────────┬─────────────────────────────┤
│               │                             │
│ Navigation    │      Active Content         │
│               │                             │
│               │                             │
├───────────────┴─────────────────────────────┤
│ Status Bar                                  │
└─────────────────────────────────────────────┘
```

The stable parts of the application remain visible:

```text
Shell
├── Header
├── Navigation
├── Content Area
└── Status Area
```

The active feature changes inside the content area.

For example:

```text
Search
   ↓
Content Area displays Search

Monitoring
   ↓
Content Area displays Monitoring

Reporting
   ↓
Content Area displays Reporting
```

### When a Shell is a good candidate

When the requirements suggest:

```text
One primary workspace
+ Shared application chrome
+ Common navigation
+ Changing feature content
```

### Important

A Shell is not automatically required because an application has 100+ screens.

The workspace and UX requirements should justify it.

---

# 5. Alternative — Multiple Windows

Another valid architecture is a multi-window application.

For example:

```text
Application
│
├── Monitoring Window
├── Reporting Window
├── Customer Window
└── Dashboard Window
```

The user may work with multiple windows simultaneously:

```text
┌────────────────────┐   ┌────────────────────┐
│ Monitoring         │   │ Reporting          │
│                    │   │                    │
│                    │   │                    │
└────────────────────┘   └────────────────────┘
```

This is appropriate when different screens represent independent workspaces.

Examples of possible scenarios:

```text
Trading
├── Market Monitor
├── Order Entry
├── Chart
└── Portfolio
```

The user may need several of these workspaces open at the same time.

## Multi-window lifecycle concerns

With multiple windows, we need to answer questions such as:

```text
Can multiple instances of the same window exist?

If a window is already open, should we:
    → Activate it?
    → Create another instance?

Who owns the window?

What happens when the main application closes?

Should the window be modal?

How do windows communicate?

How is shared state managed?
```

Therefore, a larger multi-window application may introduce a:

```text
WindowManager
```

Conceptually:

```text
Feature
   ↓
WindowManager
   ↓
Create / Find / Activate Window
```

Example:

```text
Show Monitoring
       ↓
Is Monitoring already open?
       │
       ├── Yes → Activate it
       │
       └── No → Create and show it
```

A multi-window architecture is not wrong or outdated.

It is appropriate when the UX requires multiple independent workspaces.

---

# 6. Shell vs Multiple Windows

The decision should be based primarily on the application's UX.

```text
Does the user need multiple independent workspaces
visible and usable simultaneously?
        │
        ├── Yes
        │      ↓
        │   Multiple Windows is a strong candidate
        │
        └── No
               ↓
       Is the application primarily
       one workspace with shared UI
       and changing content?
               │
               └── Yes
                      ↓
                    Shell
```

Therefore:

> **Multiple Windows are appropriate when independent workspaces are required. A Shell is appropriate when the application is primarily one workspace with stable application structure and changing feature content.**

---

# 7. Page / Frame Composition

WPF also provides built-in page-based navigation.

The structure may look like:

```text
Window
   ↓
Frame
   ↓
Page
```

Navigation can be:

```csharp
frame.Navigate(new SearchPage());
```

Conceptually:

```text
Navigation
    ↓
Frame
    ↓
SearchPage
```

The WPF navigation mechanism is naturally Page-oriented.

This can be useful when the application behaves more like a sequence of navigable pages and when built-in navigation/history behavior is useful.

However, we can still organize the application by features.

For example:

```text
Features
│
├── Search
│   ├── SearchPage
│   └── SearchViewModel
│
├── Monitoring
│   ├── MonitoringPage
│   └── MonitoringViewModel
│
└── Reporting
    ├── ReportingPage
    └── ReportingViewModel
```

---

# 8. Navigation Target: View-first vs ViewModel-first

The second architectural decision is:

> **What does the application target when activating a screen?**

---

## View-first

The navigation request identifies a View.

Example:

```csharp
NavigateTo<SearchView>();
```

Conceptually:

```text
Navigation
    ↓
SearchView
    ↓
Create / Activate View
```

The View receives its ViewModel.

For example:

```text
SearchView
     ↓
SearchViewModel
```

This is a straightforward approach when the UI element itself is the natural thing being managed.

For example, in a multi-window application:

```csharp
_windowManager.Show<MonitoringWindow>();
```

The `WindowManager` is managing an actual WPF `Window`.

This can be very natural because the Window lifecycle is UI-oriented:

```text
Create
Show
Activate
Minimize
Close
```

---

## ViewModel-first

The navigation request identifies a ViewModel.

Example:

```csharp
NavigateTo<SearchViewModel>();
```

Conceptually:

```text
Navigation
    ↓
SearchViewModel
    ↓
Resolve corresponding View
    ↓
SearchView
```

This separates two responsibilities:

```text
Navigation
    ↓
Which ViewModel becomes active?
```

and:

```text
View Resolution
    ↓
Which View represents this ViewModel?
```

A mapping mechanism is required:

```text
SearchViewModel
        ↓
SearchView
```

This can be implemented through:

```text
Naming convention
```

or:

```text
Explicit registration
```

or another View resolution mechanism.

---

# 9. Shell + View-first

This combination is possible.

```text
User action
     ↓
Navigation
     ↓
SearchView
     ↓
Shell Content Area
```

Example:

```csharp
NavigateTo<SearchView>();
```

The Shell displays the selected View.

Feature organization can still be:

```text
Features
│
├── Search
│   ├── SearchView
│   └── SearchViewModel
│
└── Reporting
    ├── ReportingView
    └── ReportingViewModel
```

---

# 10. Shell + ViewModel-first

This combination is also possible.

```text
User action
     ↓
Navigation
     ↓
SearchViewModel
     ↓
View Resolution
     ↓
SearchView
     ↓
Shell Content Area
```

Example:

```csharp
NavigateTo<SearchViewModel>();
```

The Shell may maintain something conceptually like:

```text
CurrentViewModel
```

The View resolution mechanism determines which View should display that ViewModel.

---

# 11. Multiple Windows + View-first

This is a natural combination.

```text
User requests Monitoring
        ↓
WindowManager
        ↓
MonitoringWindow
        ↓
Create / Activate
        ↓
Show()
```

Example:

```csharp
_windowManager.Show<MonitoringWindow>();
```

The Window receives its ViewModel through construction or another composition mechanism.

Conceptually:

```text
WindowManager
      ↓
MonitoringWindow
      ↓
MonitoringViewModel
```

This approach does not require a ViewModel-to-Window resolver.

---

# 12. Multiple Windows + ViewModel-first

This is also possible.

The request targets the ViewModel:

```csharp
_windowManager.Show<MonitoringViewModel>();
```

The flow becomes:

```text
WindowManager
      ↓
MonitoringViewModel
      ↓
Resolve associated Window
      ↓
MonitoringWindow
      ↓
Show / Activate
```

This requires a mapping:

```text
MonitoringViewModel
        ↓
MonitoringWindow
```

Again, the mapping can be convention-based or explicitly registered.

---

# 13. Page / Frame and Navigation

Page/Frame navigation is naturally based on a Page:

```text
Navigation
    ↓
Page
    ↓
Frame
```

Therefore, the default WPF approach is closer to a UI/Page-oriented target.

Example:

```csharp
frame.Navigate(new SearchPage());
```

However, we can build a ViewModel-oriented abstraction:

```text
NavigateTo<SearchViewModel>()
        ↓
Resolver
        ↓
SearchPage
        ↓
Frame.Navigate()
```

Therefore, Page/Frame does not prevent ViewModel-first navigation.

The difference is:

> **WPF's built-in navigation mechanism is naturally designed around Pages, while a ViewModel-first model requires an additional abstraction.**

---

# 14. Feature-Based Decomposition

This is a separate architectural concern.

The question is:

> **How do we organize a large codebase?**

A technical-layer organization may look like:

```text
Views
├── SearchView
├── MonitoringView
├── CustomerListView
└── CustomerEditView

ViewModels
├── SearchViewModel
├── MonitoringViewModel
├── CustomerListViewModel
└── CustomerEditViewModel

Services
├── SearchService
├── CustomerService
└── ReportingService
```

As the application grows, one feature becomes scattered across multiple folders.

For example:

```text
Customer Feature

Views/CustomerListView
Views/CustomerEditView

ViewModels/CustomerListViewModel
ViewModels/CustomerEditViewModel

Services/CustomerService
```

Feature-based decomposition organizes related functionality together:

```text
Features
│
├── Customers
│   ├── CustomerListView
│   ├── CustomerListViewModel
│   ├── CustomerEditView
│   ├── CustomerEditViewModel
│   └── CustomerService
│
├── Search
│   ├── SearchView
│   ├── SearchViewModel
│   └── SearchService
│
└── Reporting
    ├── ReportingView
    ├── ReportingViewModel
    └── ReportingService
```

The main idea is:

> **Organize the application primarily around business capabilities/features rather than only technical types.**

---

# 15. Feature-Based Decomposition Is Independent of Composition

Feature-based decomposition works regardless of the application composition model.

For example:

```text
Shell
+ View-first
+ Feature-based
```

```text
Shell
+ ViewModel-first
+ Feature-based
```

```text
Multiple Windows
+ View-first
+ Feature-based
```

```text
Multiple Windows
+ ViewModel-first
+ Feature-based
```

```text
Page / Frame
+ Page-oriented navigation
+ Feature-based
```

or:

```text
Page / Frame
+ ViewModel-first abstraction
+ Feature-based
```

Therefore:

> **Feature-based decomposition solves the code organization problem. Shell, Multiple Windows, and Page/Frame solve the application composition problem. View-first and ViewModel-first solve the navigation target problem.**

---

# 16. Architecture Decision Table

| Application Composition | Natural / Default Target  |  View-first |       ViewModel-first | Feature-based Decomposition |
| ----------------------- | ------------------------- | ----------: | --------------------: | --------------------------: |
| **Shell**               | Depends on implementation |           ✅ |                     ✅ |                           ✅ |
| **Multiple Windows**    | `Window`                  |           ✅ |                     ✅ |                           ✅ |
| **Page / Frame**        | `Page`                    | ✅ Naturally | ✅ Through abstraction |                           ✅ |

---

# 17. Possible Architecture Combinations

Based on these decisions, we can create multiple valid architectures.

## Combination 1

```text
Shell
+ View-first
+ Feature-based
```

## Combination 2

```text
Shell
+ ViewModel-first
+ Feature-based
```

## Combination 3

```text
Multiple Windows
+ View-first
+ Feature-based
```

## Combination 4

```text
Multiple Windows
+ ViewModel-first
+ Feature-based
```

## Combination 5

```text
Page / Frame
+ Page-oriented navigation
+ Feature-based
```

## Combination 6

```text
Page / Frame
+ ViewModel-first abstraction
+ Feature-based
```

Each combination should be evaluated against the same application requirements.

---

# 18. Navigation Is Always a Requirement, but Its Meaning Changes

Any multi-screen application needs a strategy for activating, opening, or moving between screens.

However, the strategy depends on the composition model.

## Shell

Navigation means:

```text
Which content should appear in the Shell?
```

```text
Search
    ↓
Current Content = Search
```

## Multiple Windows

Navigation/activation means:

```text
Should the requested Window:
    → Be created?
    → Be activated?
    → Be reused?
    → Allow another instance?
```

For example:

```text
Open Monitoring
       ↓
Is Monitoring already open?
       │
       ├── Yes → Activate
       │
       └── No → Create and Show
```

## Page / Frame

Navigation means:

```text
Which Page should the Frame navigate to?
```

Potential additional concerns include:

```text
Navigation history
Back
Forward
Navigation parameters
```

Therefore:

> **Navigation is the general problem of activating or moving to application functionality. `INavigationService` is only one possible implementation.**

---

# 19. Recommended Architectural Decision Process

When discussing architecture in an interview, do not start by naming patterns.

Start with requirements.

```text
Requirements
     ↓
Consider alternatives
     ↓
Evaluate trade-offs
     ↓
Choose architecture
     ↓
Explain the reasoning
```

For example:

```text
Does the user require multiple independent workspaces?

Yes
    ↓
Consider Multiple Windows

No
    ↓
Does the application have one stable workspace
with shared application chrome and changing content?

Yes
    ↓
Consider a Shell
```

Then decide how activation/navigation should work:

```text
Should the navigation system target:

View?
or
ViewModel?
```

Finally, independently decide how the application should be organized:

```text
Small/simple application
    ↓
Technical-layer organization may be sufficient

Large application
+ many business capabilities
+ 100+ screens
    ↓
Feature-based decomposition becomes a strong choice
```

---

# 20. Current Conclusion

For our architecture discussion, we have established the following:

## We have a strong reason to consider:

```text
Feature-based decomposition
```

because the application is expected to be large and contain many screens and business capabilities.

However, we should **not yet assume**:

```text
Shell
```

or:

```text
ViewModel-first
```

as universal answers.

Instead, we should evaluate:

```text
Application Composition
│
├── Shell
├── Multiple Windows
└── Page / Frame
```

and:

```text
Navigation Target
│
├── View-first
└── ViewModel-first
```

Then compare the possible combinations against the application's actual requirements.

The architectural principle is:

> **Do not choose Shell, Multiple Windows, Page/Frame, View-first, or ViewModel-first because they are considered universally correct. Choose them because they solve the requirements of the application better than the alternatives.**

## Next Step

Using the **same requirements**, implement and compare the different approaches.

For example:

```text
1. Multiple Windows
   + View-first
   + Feature-based

2. Multiple Windows
   + ViewModel-first
   + Feature-based

3. Shell
   + View-first
   + Feature-based

4. Shell
   + ViewModel-first
   + Feature-based

5. Page / Frame
   + Feature-based
```

By implementing these approaches with the same application requirements, we can directly compare:

* complexity
* coupling
* navigation responsibilities
* View/ViewModel composition
* lifecycle management
* scalability
* maintainability

Only after that comparison should we make the final architectural choice.
