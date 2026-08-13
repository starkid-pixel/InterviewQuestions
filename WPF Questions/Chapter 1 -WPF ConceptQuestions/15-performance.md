# Chapter 15 — WPF Performance

## 163. How diagnose a slow WPF application?
First identify whether the bottleneck is CPU, UI thread blocking, layout, rendering, data loading, memory pressure, or excessive allocations. Use profiling rather than optimizing based only on intuition.

## 164. What causes slow rendering?
Large visual trees, complex templates, effects, transparency, expensive drawing, animations, excessive redraws, and large images can all contribute.

## 165. What causes slow layout?
Deep/large trees, repeated invalidation, expensive measure/arrange implementations, nested panels, frequent size changes, and forced synchronous layout can make layout expensive.

## 166. Large visual tree effect
More visuals mean more objects, layout work, rendering/composition work, memory usage, and event routing overhead.

## 167. Expensive converters
Converters can run frequently during binding updates. Heavy computation, allocations, I/O, or database calls inside converters can severely degrade UI performance.

## 168. Excessive bindings
Bindings add object relationships, notifications, conversions, and update work. Thousands of unnecessarily complex bindings can become significant, especially in large item lists.

## 169. UI virtualization
It reduces the number of realized item containers and is one of the most important optimizations for large item controls.

## 170. Bitmap caching
Caching can reduce repeated rendering work for complex visuals that are expensive to redraw, but it consumes memory and is not universally beneficial.

## 171. When use CacheMode?
Use it selectively after profiling, particularly for complex visuals that are repeatedly rendered without frequent changes.

## 172. Animation performance
Animations can be efficient when they use properties that can be updated without forcing expensive layout work. Animating layout-affecting properties such as `Width` can be much more expensive than animating a transform.

## 173. Hardware acceleration
WPF can use graphics hardware for portions of rendering when supported. Hardware acceleration does not make every operation automatically fast; visual complexity and rendering choices still matter.

## 174. Diagnose high CPU
Profile CPU stacks and identify hot methods. Check layout loops, converters, binding callbacks, rendering, collection processing, polling, timers, and background work.

## 175. Diagnose high memory
Take heap snapshots, inspect object counts and retained paths, and determine whether growth is expected caching/data growth or an actual retention problem.

## 176. Diagnose UI freezes
Check UI-thread call stacks during the freeze. Common causes include synchronous I/O, CPU-heavy work, large layout operations, excessive `UpdateLayout`, huge collection changes, or lock contention.
