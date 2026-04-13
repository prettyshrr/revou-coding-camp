# Design Document: Cat Clicker App

## Overview

A single-page browser application built with plain HTML5, CSS3, and vanilla JavaScript. No build tools, no frameworks, no backend — just a single `index.html` file (or a small set of co-located static files) that can be opened directly in a browser.

The app renders clickable cat characters on screen. Each click triggers a randomly selected animation from a pool of reactions, increments per-cat and global counters, and the player can spawn up to 10 cats. The design prioritises simplicity, performance, and a playful feel.

### Key Design Decisions

- **Single HTML file or minimal file set** — keeps deployment trivial (open in browser, no server needed).
- **CSS animations over canvas** — CSS `@keyframes` are GPU-accelerated, easy to author, and require no animation loop code.
- **DOM-based cat elements** — each cat is a `<div>` with data attributes; no canvas or WebGL needed for this scale.
- **No external dependencies** — zero network requests after initial load; all assets are inline or CSS-generated.

---

## Architecture

```
index.html
├── <head>
│   └── <style>  (all CSS, including @keyframes for reactions)
└── <body>
    ├── <header>  (global counter + title)
    ├── <main id="play-field">  (cat elements live here)
    ├── <footer>  (spawn button + max-reached message)
    └── <script>  (all JS — state, event handling, animation logic)
```

### Module Boundaries (within the single script)

Even though everything lives in one `<script>` tag, the code is organised into clearly separated logical modules using plain objects / functions:

```
State          — single source of truth (cats array, globalCounter)
CatFactory     — creates cat DOM elements and assigns visual styles
ReactionEngine — selects and applies reaction animations
CounterUI      — updates counter display elements
SpawnController— manages spawn button state and cat creation
EventBus       — attaches and delegates all DOM event listeners
```

### Data Flow

```
User click / tap
      │
      ▼
EventBus.handleCatClick(catId)
      │
      ├─► State.incrementCounter(catId)
      │         │
      │         └─► CounterUI.update(catId)
      │
      └─► ReactionEngine.play(catElement)
                │
                └─► CSS class added → animation runs → class removed on animationend
```

---

## Components and Interfaces

### State

```js
// Internal shape
{
  cats: Map<string, { id, clicks, styleIndex, element }>,
  globalCounter: number   // always === sum of all cat.clicks
}

State.addCat(cat)          // registers a new cat
State.incrementCounter(id) // bumps cat clicks + globalCounter
State.getCatCount()        // returns cats.size
State.reset()              // clears everything (used on reload — automatic via page refresh)
```

### CatFactory

```js
CatFactory.create(id, styleIndex, position)
// Returns a configured <div class="cat cat--style-N"> element
// with data-id, data-clicks="0", positioned absolutely
// Contains: cat emoji/SVG, per-cat counter badge
```

Cat element structure:
```html
<div class="cat cat--style-1" data-id="cat-1" style="left: 42%; top: 38%;">
  <span class="cat__body">🐱</span>
  <span class="cat__counter">0</span>
</div>
```

### ReactionEngine

```js
ReactionEngine.play(element)
// 1. Removes any existing reaction class (handles re-click mid-animation)
// 2. Forces reflow (offsetWidth read) to restart animation
// 3. Picks a random reaction from REACTION_POOL
// 4. Adds the CSS class for that reaction
// 5. Listens for 'animationend' to remove the class
```

Reaction pool (≥ 5 types):

| Name | CSS class | Description |
|---|---|---|
| bounce | `react--bounce` | Vertical bounce up and back |
| spin | `react--spin` | 360° rotation |
| shake | `react--shake` | Rapid horizontal oscillation |
| heartburst | `react--heartburst` | Scales up with a ❤️ overlay that fades |
| pulse | `react--pulse` | Scale up then back to normal |
| wobble | `react--wobble` | Rotation oscillation (rubber feel) |

### CounterUI

```js
CounterUI.updateCat(id, value)   // sets .cat__counter text for cat `id`
CounterUI.updateGlobal(value)    // sets #global-counter text
```

### SpawnController

```js
SpawnController.spawn()
// Guards: if cats.size >= MAX_CATS → no-op
// Calls CatFactory.create, State.addCat, appends to #play-field
// Updates spawn button disabled state + max message visibility

SpawnController.updateUI()
// Disables button and shows message when cats.size === MAX_CATS
```

### EventBus

```js
EventBus.init()
// Attaches delegated click+touchstart listener on #play-field
// Attaches click listener on spawn button
// Uses event.target.closest('.cat') to identify cat clicks
```

---

## Data Models

### Cat Record (in-memory only, no persistence)

```ts
interface CatRecord {
  id: string;          // e.g. "cat-1", "cat-2"
  clicks: number;      // per-cat click count, starts at 0
  styleIndex: number;  // 0–4, determines visual variant
  element: HTMLElement;
}
```

### App State

```ts
interface AppState {
  cats: Map<string, CatRecord>;
  globalCounter: number;
}
```

### Reaction

```ts
interface Reaction {
  name: string;
  cssClass: string;
}

const REACTION_POOL: Reaction[] = [
  { name: 'bounce',     cssClass: 'react--bounce'     },
  { name: 'spin',       cssClass: 'react--spin'       },
  { name: 'shake',      cssClass: 'react--shake'      },
  { name: 'heartburst', cssClass: 'react--heartburst' },
  { name: 'pulse',      cssClass: 'react--pulse'      },
  { name: 'wobble',     cssClass: 'react--wobble'     },
];
```

### Visual Style Variants

```ts
// 5 distinct styles applied via CSS class
const CAT_STYLES = [
  { cssClass: 'cat--style-1', emoji: '🐱', hue: 'orange' },
  { cssClass: 'cat--style-2', emoji: '😸', hue: 'pink'   },
  { cssClass: 'cat--style-3', emoji: '🐈', hue: 'teal'   },
  { cssClass: 'cat--style-4', emoji: '😺', hue: 'purple' },
  { cssClass: 'cat--style-5', emoji: '🐈‍⬛', hue: 'dark'  },
];
```

Adjacent-cat style uniqueness is enforced at spawn time: the new cat's `styleIndex` is chosen by cycling through styles and skipping the index used by the most recently added cat.

### Position

Cats are positioned absolutely within `#play-field` using percentage-based `left` / `top` values (clamped to keep the cat fully within bounds, accounting for cat size as a percentage of viewport).

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Counter invariant

*For any* sequence of clicks distributed across any number of cats, the global counter SHALL always equal the arithmetic sum of all individual cat click counters, and each individual cat counter SHALL equal the number of times that cat was clicked.

**Validates: Requirements 3.1, 3.3**

### Property 2: Reaction is always from the pool

*For any* cat click, the reaction animation class applied to the cat element SHALL correspond to exactly one entry in REACTION_POOL and no other value.

**Validates: Requirements 2.1**

### Property 3: Reaction restarts on re-click

*For any* cat that currently has a reaction CSS class applied, clicking it again SHALL remove the existing class, force a reflow, and re-add a reaction class — resetting the animation timeline to the beginning.

**Validates: Requirements 2.5**

### Property 4: Reaction cleanup after completion

*For any* reaction animation applied to a cat, once the `animationend` event fires (or the fallback timeout elapses), the cat element SHALL have no `react--*` CSS class present.

**Validates: Requirements 2.4**

### Property 5: Spawn increases cat count by exactly one

*For any* app state where the current cat count is less than MAX_CATS, triggering the spawn action SHALL increase the cat count by exactly 1 and the newly added cat SHALL respond to click events immediately.

**Validates: Requirements 4.2, 4.5**

### Property 6: Cat count never exceeds maximum

*For any* number of spawn attempts (including attempts beyond the limit), the total number of cats on screen SHALL never exceed MAX_CATS (10).

**Validates: Requirements 4.3**

### Property 7: Style assignment validity and adjacency

*For any* sequence of cat spawns, each cat SHALL be assigned a styleIndex in the range [0, CAT_STYLES.length − 1], and no two consecutively spawned cats SHALL share the same styleIndex.

**Validates: Requirements 5.1, 5.2**

### Property 8: Cat style is immutable after assignment

*For any* cat and any number of clicks applied to it, the cat's styleIndex SHALL remain identical to the value assigned at creation time.

**Validates: Requirements 5.3**

### Property 9: Cats remain within viewport bounds

*For any* viewport width in the supported range [320px, 2560px], the position computation function SHALL return `left` and `top` percentage values such that the cat element is fully contained within the play-field area with no overflow.

**Validates: Requirements 1.2, 1.3, 6.1**

---

## Error Handling

| Scenario | Handling |
|---|---|
| Spawn called when at max cats | Guard in `SpawnController.spawn()` — silently no-ops; UI already shows disabled state |
| Click on non-cat element | `event.target.closest('.cat')` returns null → handler returns early |
| `animationend` never fires (browser quirk) | Reaction class is also removed after a fixed timeout fallback (duration + 50ms) |
| Invalid cat ID in state lookup | Defensive check; logs a warning to console, no crash |
| Touch + click double-fire | `touchstart` handler calls `event.preventDefault()` to suppress the synthetic mouse click |

---

## Testing Strategy

### Unit Tests (example-based)

Focus on pure logic functions that have no DOM dependency:

- `selectReaction()` — returns a valid member of REACTION_POOL
- `computePosition(viewportW, viewportH, catSize)` — returns clamped `{left, top}` percentages
- `pickStyle(lastStyleIndex, styleCount)` — never returns `lastStyleIndex`
- `State.incrementCounter(id)` — counter values update correctly
- `State.getCatCount()` — returns correct size

### Property-Based Tests

Using [fast-check](https://github.com/dubzzz/fast-check) (JavaScript PBT library). Each test runs a minimum of 100 iterations.

**Property 1 — Counter invariant**
```js
// Feature: cat-clicker-app, Property 1: counter invariant
fc.assert(fc.property(
  fc.array(fc.nat({ max: 9 }), { minLength: 1, maxLength: 50 }),
  (clickSequence) => {
    // simulate clicks on cats by index
    // verify each cat.clicks === number of times that index appeared
    // verify globalCounter === sum(cat.clicks)
  }
), { numRuns: 100 });
```

**Property 2 — Reaction always from pool**
```js
// Feature: cat-clicker-app, Property 2: reaction is always from the pool
fc.assert(fc.property(
  fc.integer({ min: 0, max: 1000 }),
  (seed) => {
    const reaction = selectReaction(seed);
    return REACTION_POOL.some(r => r.cssClass === reaction.cssClass);
  }
), { numRuns: 100 });
```

**Property 5 — Spawn increases count by exactly one**
```js
// Feature: cat-clicker-app, Property 5: spawn increases cat count by exactly one
fc.assert(fc.property(
  fc.integer({ min: 0, max: 9 }),
  (initialCount) => {
    // set up state with initialCount cats (< MAX_CATS)
    // call spawn once, verify cats.size === initialCount + 1
    // verify new cat responds to click (counter increments)
  }
), { numRuns: 100 });
```

**Property 6 — Cat count never exceeds maximum**
```js
// Feature: cat-clicker-app, Property 6: cat count never exceeds maximum
fc.assert(fc.property(
  fc.integer({ min: 11, max: 50 }),
  (spawnAttempts) => {
    // attempt to spawn N > MAX_CATS cats, verify cats.size <= MAX_CATS
  }
), { numRuns: 100 });
```

**Property 7 — Style assignment validity and adjacency**
```js
// Feature: cat-clicker-app, Property 7: style assignment validity and adjacency
fc.assert(fc.property(
  fc.integer({ min: 2, max: 10 }),
  (catCount) => {
    // spawn N cats, for each cat verify styleIndex in [0, CAT_STYLES.length-1]
    // verify no two consecutive cats share styleIndex
  }
), { numRuns: 100 });
```

**Property 8 — Style immutability**
```js
// Feature: cat-clicker-app, Property 8: cat style is immutable after assignment
fc.assert(fc.property(
  fc.nat({ max: 200 }),
  (clickCount) => {
    // create a cat, record its styleIndex
    // click it N times, verify styleIndex is unchanged
  }
), { numRuns: 100 });
```

**Property 9 — Cats remain within viewport bounds**
```js
// Feature: cat-clicker-app, Property 9: cats remain within viewport bounds
fc.assert(fc.property(
  fc.integer({ min: 320, max: 2560 }),
  fc.integer({ min: 400, max: 1440 }),
  (viewportW, viewportH) => {
    const pos = computePosition(viewportW, viewportH, CAT_SIZE_PX);
    // verify pos.left and pos.top keep cat fully within bounds
    return pos.left >= 0 && pos.top >= 0
      && pos.left + CAT_SIZE_PX <= viewportW
      && pos.top + CAT_SIZE_PX <= viewportH;
  }
), { numRuns: 100 });
```

### Integration / Smoke Tests (manual or with a headless browser)

- Page loads without JS errors (smoke)
- Spawn button disables at 10 cats (example)
- Touch events trigger reactions on mobile viewport (example)
- All cats remain in bounds after viewport resize (example)
- Reaction animation class is removed after `animationend` (example)

### Performance Checks

- Measure `requestAnimationFrame` callback timing with 10 cats all animating simultaneously — assert ≥ 30 fps
- Measure page load time with `performance.timing` — assert ≤ 3 seconds
