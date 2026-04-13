# Implementation Plan: Cat Clicker App

## Overview

Build a single-page cat clicker app as a minimal set of static files (HTML, CSS, vanilla JS). Implementation follows the module boundaries defined in the design: State, CatFactory, ReactionEngine, CounterUI, SpawnController, and EventBus — all within a single `index.html` file.

## Tasks

- [x] 1. Set up project structure and core constants
  - Create `index.html` with the HTML skeleton: `<header>`, `<main id="play-field">`, `<footer>`, and an inline `<script>` tag
  - Define `REACTION_POOL`, `CAT_STYLES`, and `MAX_CATS` constants
  - Add a `<style>` block with base layout CSS (header, play-field, footer, spawn button)
  - _Requirements: 1.1, 1.4, 3.4, 4.1_

- [x] 2. Implement State module and pure logic helpers
  - [x] 2.1 Implement the `State` module
    - Write `State` as a plain object with `cats` (`Map`), `globalCounter`, `addCat`, `incrementCounter`, `getCatCount`, and `reset`
    - _Requirements: 3.1, 3.3_

  - [ ]* 2.2 Write property test for counter invariant (Property 1)
    - **Property 1: Counter invariant**
    - **Validates: Requirements 3.1, 3.3**
    - Use fast-check: generate random click sequences across cats, assert each cat's click count equals appearances in sequence and `globalCounter` equals the sum

  - [x] 2.3 Implement `computePosition(viewportW, viewportH, catSizePx)` helper
    - Returns `{ left, top }` percentage values clamped so the cat stays fully within the play-field
    - _Requirements: 1.2, 1.3, 6.1_

  - [ ]* 2.4 Write property test for viewport bounds (Property 9)
    - **Property 9: Cats remain within viewport bounds**
    - **Validates: Requirements 1.2, 1.3, 6.1**
    - Use fast-check: generate viewport widths [320, 2560] and heights [400, 1440], assert returned position keeps cat fully in bounds

  - [x] 2.5 Implement `pickStyle(lastStyleIndex, styleCount)` helper
    - Returns a styleIndex in `[0, styleCount − 1]` that is never equal to `lastStyleIndex`
    - _Requirements: 5.1, 5.2_

  - [ ]* 2.6 Write property test for style assignment validity and adjacency (Property 7)
    - **Property 7: Style assignment validity and adjacency**
    - **Validates: Requirements 5.1, 5.2**
    - Use fast-check: generate sequences of spawn counts [2, 10], assert each styleIndex is in range and no two consecutive cats share the same index

  - [x] 2.7 Implement `selectReaction()` helper
    - Picks a random entry from `REACTION_POOL` and returns it
    - _Requirements: 2.1_

  - [ ]* 2.8 Write property test for reaction always from pool (Property 2)
    - **Property 2: Reaction is always from the pool**
    - **Validates: Requirements 2.1**
    - Use fast-check: generate integer seeds, assert `selectReaction()` always returns a member of `REACTION_POOL`

- [~] 3. Checkpoint — Ensure all pure logic tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [~] 4. Implement CatFactory and visual styles
  - [~] 4.1 Implement `CatFactory.create(id, styleIndex, position)`
    - Creates and returns a `<div class="cat cat--style-N">` element with `data-id`, `data-clicks="0"`, absolute positioning, a `.cat__body` emoji span, and a `.cat__counter` badge
    - _Requirements: 1.4, 5.1, 5.3_

  - [~] 4.2 Add CSS for cat variants and layout
    - Write `cat--style-1` through `cat--style-5` rules (distinct emoji + background hue per `CAT_STYLES`)
    - Add responsive sizing: on viewports < 768px scale cats so at least 2 fit without scrolling
    - _Requirements: 1.4, 5.1, 6.1, 6.2_

  - [ ]* 4.3 Write property test for style immutability (Property 8)
    - **Property 8: Cat style is immutable after assignment**
    - **Validates: Requirements 5.3**
    - Use fast-check: create a cat, record its `styleIndex`, simulate N clicks, assert `styleIndex` is unchanged

- [~] 5. Implement ReactionEngine and reaction CSS
  - [~] 5.1 Add `@keyframes` for all 6 reactions
    - Write `@keyframes` for `react--bounce`, `react--spin`, `react--shake`, `react--heartburst`, `react--pulse`, `react--wobble`
    - Each animation must complete within a defined duration (e.g. 400–600ms)
    - _Requirements: 2.2, 2.3, 7.1, 7.2_

  - [~] 5.2 Implement `ReactionEngine.play(element)`
    - Remove any existing `react--*` class, force reflow via `element.offsetWidth`, pick a reaction with `selectReaction()`, add the CSS class, listen for `animationend` to remove it, and set a fallback timeout (duration + 50ms) as a safety net
    - _Requirements: 2.1, 2.2, 2.4, 2.5_

  - [ ]* 5.3 Write property test for reaction restart on re-click (Property 3)
    - **Property 3: Reaction restarts on re-click**
    - **Validates: Requirements 2.5**
    - Verify that calling `ReactionEngine.play` on an element that already has a `react--*` class removes the old class, forces reflow, and re-adds a reaction class

  - [ ]* 5.4 Write property test for reaction cleanup after completion (Property 4)
    - **Property 4: Reaction cleanup after completion**
    - **Validates: Requirements 2.4**
    - Verify that after `animationend` fires (or fallback timeout elapses), no `react--*` class remains on the element

- [~] 6. Implement CounterUI and SpawnController
  - [~] 6.1 Implement `CounterUI.updateCat(id, value)` and `CounterUI.updateGlobal(value)`
    - Update the `.cat__counter` badge text for the given cat and the `#global-counter` element in the header
    - _Requirements: 3.2, 3.4_

  - [~] 6.2 Implement `SpawnController.spawn()` and `SpawnController.updateUI()`
    - `spawn()`: guard against `cats.size >= MAX_CATS`, call `CatFactory.create`, `State.addCat`, append to `#play-field`, call `updateUI()`
    - `updateUI()`: disable spawn button and show max-reached message when `cats.size === MAX_CATS`
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

  - [ ]* 6.3 Write property test for spawn increases count by exactly one (Property 5)
    - **Property 5: Spawn increases cat count by exactly one**
    - **Validates: Requirements 4.2, 4.5**
    - Use fast-check: for initial counts [0, 9], call `spawn()` once, assert `cats.size` increased by exactly 1 and new cat's counter increments on click

  - [ ]* 6.4 Write property test for cat count never exceeds maximum (Property 6)
    - **Property 6: Cat count never exceeds maximum**
    - **Validates: Requirements 4.3**
    - Use fast-check: attempt N > MAX_CATS spawns, assert `cats.size <= MAX_CATS` at all times

- [~] 7. Implement EventBus and wire everything together
  - [~] 7.1 Implement `EventBus.init()`
    - Attach a delegated `click` and `touchstart` listener on `#play-field` using `event.target.closest('.cat')` to identify cat clicks
    - On cat click: call `State.incrementCounter(id)`, `CounterUI.updateCat(id, value)`, `CounterUI.updateGlobal(value)`, `ReactionEngine.play(element)`
    - Call `event.preventDefault()` in `touchstart` to suppress double-fire
    - Attach `click` listener on the spawn button to call `SpawnController.spawn()`
    - _Requirements: 2.1, 2.2, 3.1, 3.3, 6.3_

  - [~] 7.2 Bootstrap the app on `DOMContentLoaded`
    - Call `EventBus.init()`, spawn the initial cat via `SpawnController.spawn()`, and call `CounterUI.updateGlobal(0)`
    - _Requirements: 1.1, 3.4_

- [~] 8. Final checkpoint — Ensure all tests pass and app runs without console errors
  - Ensure all tests pass, ask the user if questions arise.
  - Verify no JavaScript errors appear in the browser console during normal interaction (_Requirements: 7.4_)

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Property tests use [fast-check](https://github.com/dubzzz/fast-check) with a minimum of 100 iterations each
- All code lives in a single `index.html` (or minimal co-located static files) — no build step required
- Each task references specific requirements for traceability
