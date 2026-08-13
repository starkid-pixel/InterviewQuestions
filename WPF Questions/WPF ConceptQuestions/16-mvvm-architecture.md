# Chapter 16 — MVVM and Application Architecture

## 177. What belongs in a ViewModel?
Presentation state, commands, validation state, user-facing operations, and coordination of application services needed by the View. It should expose data in a way the View can bind to.

## 178. What should not belong in a ViewModel?
Concrete control manipulation, visual-tree traversal, direct Window ownership, and UI-specific drawing details generally do not belong there. Abstract UI services when necessary.

## 179. Is code-behind forbidden in MVVM?
No. MVVM is about separation of responsibilities, not banning every line of code-behind. View-specific behavior can reasonably live in code-behind when moving it to a ViewModel would create worse coupling.

## 180. When is code-behind acceptable?
Purely visual behavior, focus management, animation setup, control-specific event handling, and interactions that do not represent business/presentation state are common examples.

## 181. How handle View-specific behavior?
Use attached behaviors, custom controls, interaction behaviors, or carefully scoped code-behind. Choose the simplest mechanism that preserves the intended separation.

## 182. How communicate between View and ViewModel?
Normally through bindings and commands. For specialized interactions, use services, behaviors, or request/response abstractions.

## 183. How communicate between ViewModels?
Prefer explicit service interfaces or parent/child coordination. Messaging is useful for decoupled cross-cutting notifications but should not become an untraceable global dependency system.

## 184. How handle navigation?
Introduce a navigation service or navigation abstraction. ViewModels request navigation without knowing concrete Window/Page implementation details.

## 185. How handle dialogs?
Use a dialog service abstraction that can be mocked in tests.

## 186. How handle application-wide state?
Use an application state/service abstraction with clear ownership and lifetime. Avoid uncontrolled global mutable state.

## 187. How inject dependencies into ViewModels?
Use constructor injection through a dependency-injection container or composition root.

## 188. How unit test ViewModels?
Test commands, state changes, validation, service calls, and notifications using fake/mock services. Avoid requiring a live Window or UI thread for ordinary ViewModel tests.

## 189. How prevent God ViewModels?
Split by feature/responsibility, extract services, use child ViewModels for complex views, and keep domain/application logic out of presentation classes.
