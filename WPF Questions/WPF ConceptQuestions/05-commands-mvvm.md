# Chapter 5 — Commands and MVVM

## 54. What problem does ICommand solve?
`ICommand` separates an operation from a particular UI event. A Button can execute a command without knowing the ViewModel implementation, improving testability and reuse.

## 55. Execute and CanExecute
`Execute` performs the operation. `CanExecute` reports whether the operation is currently allowed.

## 56. What is CanExecuteChanged?
It tells command consumers that the result of `CanExecute` may have changed, so controls should reevaluate whether the command is enabled.

## 57. Why can a Button remain disabled?
The command may not raise `CanExecuteChanged`, or the implementation may rely on `CommandManager` and not trigger requery when expected. Explicitly raise the event or use an appropriate command implementation.

## 58. What is CommandManager.RequerySuggested?
WPF's `CommandManager` provides a mechanism for reevaluating command states when relevant UI conditions change. It can reduce the need for manual `CanExecuteChanged` handling, but custom command implementations should still be designed carefully.

## 59. Why call InvalidateRequerySuggested?
It asks WPF to requery command states. It can be useful when application state changed but WPF has no reason to automatically perform a command requery.

## 60. ICommand vs event handlers
Events directly couple a View's event to a method. Commands expose an operation and its enabled state as an abstraction, making it easier to bind from XAML and test independently of a particular control.

## 61. What is RelayCommand/DelegateCommand?
It is a small command implementation that delegates `Execute` and `CanExecute` to supplied methods/delegates. It is common in MVVM libraries and custom ViewModels.

## 62. Should ViewModels reference Views?
Ideally, ViewModels should not depend directly on concrete Views. If a ViewModel requires UI services, abstract the capability behind an interface such as `IDialogService` or `INavigationService`.

## 63. How do you open a Window from a ViewModel?
Use a dialog/window service interface injected into the ViewModel, or a navigation abstraction. The ViewModel requests an operation; the View/service performs the actual UI work.

## 64. How do you close a Window from a ViewModel?
Use an injected window/navigation service, a close-request message, or a framework command mechanism. Avoid making the ViewModel directly call `Window.Close()`.

## 65. How do ViewModels communicate?
Use explicit service abstractions, parent-child coordination, messages/event aggregators, or shared application state. Prefer clear ownership over global messaging everywhere.

## 66. What is a messenger/event aggregator?
It provides decoupled publish/subscribe communication. Components publish messages without directly referencing each subscriber.

## 67. How do you avoid event memory leaks?
Unsubscribe when appropriate, use weak-event patterns where suitable, or use lifetime-aware messaging. The key is preventing a long-lived publisher from retaining a short-lived subscriber.

### Scenario: ViewModel needs to show a dialog
Define something like `IDialogService.ShowConfirmation(...)`. Inject it into the ViewModel. Unit tests can supply a fake implementation. The concrete WPF service can call `MessageBox` or a custom dialog Window.
