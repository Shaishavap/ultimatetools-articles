# Building 5 Classic Browser Games in React — Canvas RAF, Pointer Events Drag & Drop, No Game Libraries

Five games. Five different architectural patterns. All client-side React, no game libraries, no physics engines, no canvas frameworks.

Here's a technical overview of how [Snake](https://ultimatetools.io/tools/fun-tools/snake/), [Minesweeper](https://ultimatetools.io/tools/fun-tools/minesweeper/), [Brick Breaker](https://ultimatetools.io/tools/fun-tools/brick-breaker/), [Sudoku](https://ultimatetools.io/tools/fun-tools/sudoku/), and [Solitaire](https://ultimatetools.io/tools/fun-tools/solitaire/) are built — what each one taught about browser game development, and why the approach differs between them.

---

## The Two Architectures

All five games fall into one of two categories:

**Canvas games (RAF loop):** Snake, Brick Breaker
- State lives in refs, never in React state
- `requestAnimationFrame` + `setTimeout` for a fixed tick rate
- Canvas API handles all rendering — React only mounts the `<canvas>` element
- Pointer/keyboard events write to refs directly

**DOM games (React state):** Minesweeper, Sudoku, Solitaire
- State lives in `useState` / `useReducer`
- React re-renders handle all rendering
- Interactions trigger state updates, React diffs and repaints
- CSS handles animations (transitions, transforms)

The choice comes down to one question: **does the game need 60fps continuous rendering?**

If yes → Canvas + RAF. If no → React state + DOM.

---

## Snake — All State in Refs

The core insight for Snake: `requestAnimationFrame` callbacks capture stale closures. If your game state is in `useState`, the RAF callback sees the state from when it was registered — not the current state.

Solution: everything goes in `useRef`.

```typescript
const snakeRef = useRef<Segment[]>([{ x: 10, y: 10 }])
const dirRef = useRef<Direction>('right')
const dirQueueRef = useRef<Direction[]>([])
const foodRef = useRef<Segment>(randomFood([]))
const aliveRef = useRef(true)
```

The RAF loop reads and mutates refs directly. Only `score` and `alive` live in React state because they need to trigger UI re-renders (score display, game-over screen).

**Direction queue** buffers up to 2 keypresses between ticks — prevents the double-reversal bug when a player presses two keys quickly.

**Speed scaling:** `setTimeout(fn, Math.max(60, 150 - floor(score/50) * 10))` inside RAF. Starts at 150ms/tick, minimum 60ms. No level system needed.

[Full breakdown → Building Snake in React with Canvas + RAF](https://dev.to/shaishav_patel_271fdcd61a/building-snake-in-react-with-canvas-requestanimationframe-all-state-in-refs-no-stale-closures-1f9)

---

## Brick Breaker — AABB Collision Physics

Brick Breaker uses the same Canvas + RAF + refs pattern as Snake, but adds continuous physics.

The ball has position `(x, y)` and velocity `(vx, vy)`. Each frame:
1. Advance position by velocity
2. Check wall/ceiling collisions → reflect velocity
3. Check paddle collision → recalculate angle from hit position
4. Check all brick collisions → AABB test, reflect on dominant axis

**The AABB collision trick:** when the ball hits a brick corner, determine which axis to reflect by comparing the overlap on each axis:

```typescript
const overlapX = (brick.width / 2) - Math.abs(ball.x - brickCenterX)
const overlapY = (brick.height / 2) - Math.abs(ball.y - brickCenterY)

if (overlapX < overlapY) ball.vx *= -1
else ball.vy *= -1
```

The smaller overlap = the axis the ball entered from. This handles corners correctly without tunneling.

**Paddle angle control:** hit position mapped to `±67.5°`. Dead center = straight up. Edges = shallow angle. Makes the game strategic, not random.

**Power-ups** fall from destroyed bricks and are caught by the paddle. Applied via direct ref mutation, timers via `setTimeout` — no React state needed since the game loop reads refs every frame.

[Full breakdown → Building Brick Breaker in React](https://dev.to/shaishav_patel_271fdcd61a/building-brick-breaker-in-react-ball-physics-aabb-collision-and-power-ups-on-canvas-2jn)

---

## Minesweeper — BFS Flood Fill

Minesweeper is a DOM game — cells render as a CSS grid, React state handles everything.

**Safe first click:** mines are placed *after* the first click, excluding the clicked cell and its 8 neighbors. Guarantees a meaningful opening reveal.

**BFS cascade reveal:** when a zero-cell (no adjacent mines) is clicked, flood-fill outward:

```typescript
const queue: [number, number][] = [[startRow, startCol]]
while (queue.length > 0) {
  const [r, c] = queue.shift()!
  cell.isRevealed = true
  if (cell.adjacentMines === 0) {
    // push all unrevealed, unflagged neighbors
  }
}
```

BFS (not DFS recursion) avoids stack overflow on large boards and reveals cells in outward waves, matching visual intuition.

**Responsive scale-to-fit:** Expert difficulty is 30×16 cells — 960px wide at 32px/cell. On mobile, a `ResizeObserver` measures the container width and applies `transform: scale(containerW / boardW)`. The board shrinks to fit without reflowing or changing cell sizes. `marginBottom` compensation closes the layout gap left by the shrunken element.

[Full breakdown → Building Minesweeper in React](https://ultimatetools.hashnode.dev/building-minesweeper-in-react-bfs-flood-fill-safe-first-click-and-responsive-scale-to-fit)

---

## Sudoku — Backtracking Algorithm

Sudoku has two algorithmic challenges: generating a valid solved board, and verifying the puzzle has a unique solution.

**Solver:** standard backtracking — try each number 1–9 in the first empty cell, recurse, backtrack on failure.

**Puzzle generation:**
1. Fill a grid with a *randomized* solver (shuffle number order before trying) → different solved board each time
2. Remove cells one at a time
3. After each removal, run `countSolutions()` — if count > 1, restore the cell
4. Stop when the target clue count is reached (Easy: 45, Medium: 35, Hard: 26)

**Uniqueness check stops at 2** — we don't need the total count, just whether it's exactly 1:

```typescript
function countSolutions(grid: Grid, limit = 2): number {
  // ... backtrack ...
  if (count >= limit) return count // early exit
}
```

This keeps generation fast (~50–200ms) instead of exhaustively counting all solutions.

**Notes mode** stores pencil marks as `Set<number>` per cell — fast add/delete/has. When a digit is entered, clear that number from all notes in the same row, column, and box automatically.

[Full breakdown → Building a Sudoku Game in React](https://dev.to/shaishav_patel_271fdcd61a/building-a-sudoku-game-in-react-backtracking-solver-puzzle-generation-notes-mode-and-hints-b5j)

---

## Solitaire — Pointer Events Drag & Drop

Solitaire is the most complex: 52 cards, 5 zones, multi-card drags, move validation, auto-complete.

**Why not a drag-and-drop library:** dnd-kit and react-dnd are built for lists and kanban. Solitaire needs cards that stack at an angle, drag as a unit (multiple cards), and drop on tiny exposed edges. Pointer Events API gave direct control without fighting library abstractions.

**`setPointerCapture`** is the critical call:

```typescript
e.currentTarget.setPointerCapture(e.pointerId)
```

Without it, `pointermove` fires on whatever element the pointer is over — which during a fast drag is often not the card. With capture, all events route to the capturing element until `pointerup`, regardless of where the pointer moves.

**Drop detection via `elementFromPoint`:**

```typescript
const target = document.elementFromPoint(e.clientX, e.clientY)
const dropZone = target?.closest('[data-drop-zone]')
```

Each valid target has `data-drop-zone` and `data-col` attributes. Hit detection on release, not during drag — no continuous zone tracking needed.

**Move validation:**
- Tableau: alternating color + descending rank. Empty column accepts only Kings.
- Foundation: same suit + ascending rank. Empty pile accepts only Aces.

**Auto-complete:** when all tableau cards are face-up, scan waste + columns for any card that can move to foundation. Run in a 200ms interval for a satisfying card-by-card animation.

[Full breakdown → Building Klondike Solitaire in React](https://ultimatetools.hashnode.dev/building-klondike-solitaire-in-react-pointer-events-drag-drop-without-a-library)

---

## What Each Game Taught

| Game | Key lesson |
|---|---|
| Snake | Refs solve stale closure in RAF. Direction queue prevents input loss. |
| Brick Breaker | AABB overlap comparison handles corners. Paddle angle = hit position mapping. |
| Minesweeper | BFS > DFS recursion for cascade reveal. ResizeObserver scale-to-fit for fixed-grid responsiveness. |
| Sudoku | Backtracking + uniqueness check. `countSolutions(limit=2)` for fast generation. |
| Solitaire | `setPointerCapture` + `elementFromPoint` replaces entire DnD library. |

No canvas framework, no physics engine, no game library. Five different problems, five targeted solutions using browser APIs directly.

---

*All games are free and run in your browser: [Ultimate Tools](https://ultimatetools.io/tools/fun-tools/)*