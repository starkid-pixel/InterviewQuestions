# Chapter 13 — Layout and Rendering

## 144. Explain WPF layout.
WPF normally performs a measure pass followed by an arrange pass. Elements determine desired size during measurement and receive final layout slots during arrangement. Rendering then draws the resulting visuals.

## 145. What happens during Measure?
A parent gives a child an available-size constraint. The child measures its content and reports `DesiredSize`.

## 146. What happens during Arrange?
The parent assigns the child its final layout rectangle. The child establishes its final size/position and arranges descendants.

## 147. ActualWidth vs Width
`Width` is the requested/layout value and may be `Auto`/unset. `ActualWidth` is the resulting rendered/layout width after measurement and arrangement.

## 148. DesiredSize and RenderSize
`DesiredSize` is the size requested during measurement. `RenderSize` is the final size allocated during arrange.

## 149. What causes layout invalidation?
Changes to layout-affecting properties, child additions/removals, parent constraints, templates, margins, alignment, and similar changes can invalidate measurement or arrangement.

## 150. What is InvalidateMeasure?
It marks an element as needing a new measure pass.

## 151. What is InvalidateArrange?
It marks an element as needing a new arrange pass.

## 152. What is InvalidateVisual?
It indicates that the visual should be redrawn without necessarily requiring a full layout pass.

## 153. Layout vs rendering
Layout determines geometry and positions. Rendering draws the resulting visual content. A visual change can require rendering without changing layout; a size change can require both.

## 154. Why can UpdateLayout hurt performance?
It forces layout processing synchronously instead of letting WPF batch/coalesce changes. Repeated calls can cause excessive layout work and UI stalls.
