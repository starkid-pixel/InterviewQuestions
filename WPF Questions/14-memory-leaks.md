# Chapter 14 — Memory Leaks

## 155. Can WPF applications leak memory despite GC?
Yes. Garbage collection only collects objects that are no longer reachable. Long-lived references can unintentionally keep objects alive.

## 156. How do event handlers cause leaks?
If a long-lived publisher stores a delegate referencing a short-lived subscriber, the publisher keeps the subscriber reachable. The subscriber cannot be collected until the subscription is removed or the publisher dies.

## 157. How can static events leak?
Static events are effectively long-lived. A subscription from a short-lived object can therefore keep that object alive for the application's lifetime.

## 158. How can timers leak?
A timer can hold references to its callback target. If the timer continues running, the target can remain reachable.

## 159. DispatcherTimer lifetime issue
`DispatcherTimer` runs through the UI Dispatcher and can keep its target reachable through the event handler. Stop/dispose appropriate resources and unsubscribe when the owning View is no longer needed.

## 160. ViewModel event leaks
A long-lived service or event aggregator subscribing to a short-lived ViewModel can retain that ViewModel. Use explicit unsubscription or lifetime-aware/weak subscriptions.

## 161. What is WeakEventManager?
It implements a weak event pattern so event subscriptions do not necessarily keep listeners alive. It is useful for certain long-lived publishers/listeners.

## 162. How investigate a WPF memory leak?
Take memory snapshots at repeated lifecycle points, compare retained objects and reference paths, and identify the root object keeping a supposedly dead Window/ViewModel alive. Tools such as Visual Studio diagnostics, dotMemory, or PerfView can help.

### Scenario
Open/close the same Window repeatedly, take snapshots after GC, compare instance counts, and inspect retention paths. Look especially for static events, service subscriptions, timers, dispatcher callbacks, event aggregators, and global collections.
