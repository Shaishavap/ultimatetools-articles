# Building Snake in React with Canvas + requestAnimationFrame — All State in Refs, No Stale Closures

Snake is a deceptively simple game to describe and surprisingly tricky to implement correctly in React. The main trap: `useState` and `requestAnimationFrame` don't mix well. The RAF callback captures a stale closure on the state from when it was created. By the time your game loop runs, the state it sees is outdated.

Here's how I built the [Snake game](https://ultimatetools.io/tools/fun-tools/snake/) using Canvas, RAF, and refs to sidestep the stale closure problem entirely.

## The Stale Closure Problem

Consider this naive approach:

```typescript
const [snake, setSnake] = useState([{ x: 10, y: 10 }])

useEffect(() => {
  const loop = () => {
    // ❌ `snake` is the value from when this effect ran — always stale
    const head = snake[0]
    const newHead = { x: head.x + 1, y: head.y }
    setSnake([newHead, ...snake.slice(0, -1)])
    requestAnimationFrame(loop)
  }
  requestAnimationFrame(loop)
}, []) // no deps = stale forever
```

The fix: store all mutable game state in refs. Refs don't trigger re-renders, they're always current, and RAF callbacks read them directly.

## State Structure in Refs

```typescript
const GRID = 20  // 20x20 cells
const CELL = 20  // pixels per cell

interface Segment { x: number; y: number }

// All game state lives in refs
const snakeRef = useRef<Segment[]>([{ x: 10, y: 10 }])
const dirRef = useRef<'up' | 'down' | 'left' | 'right'>('right')
const dirQueueRef = useRef<typeof dirRef.current[]>([])
const foodRef = useRef<Segment>(randomFood(snakeRef.current))
const scoreRef = useRef(0)
const aliveRef = useRef(true)
const rafRef = useRef<number>(0)

// Only score and alive state need to trigger re-renders
const [score, setScore] = useState(0)
const [alive, setAlive] = useState(true)
```

The pattern: refs for everything the game loop reads/writes, state only for things the React UI needs to display.

## The Game Loop

```typescript
const canvasRef = useRef<HTMLCanvasElement>(null)

function gameLoop() {
  if (!aliveRef.current) return

  const canvas = canvasRef.current!
  const ctx = canvas.getContext('2d')!

  // Process direction queue
  if (dirQueueRef.current.length > 0) {
    const next = dirQueueRef.current.shift()!
    if (!isOpposite(next, dirRef.current)) {
      dirRef.current = next
    }
  }

  // Move snake
  const head = snakeRef.current[0]
  const newHead = moveHead(head, dirRef.current)

  // Wall collision
  if (newHead.x < 0 || newHead.x >= GRID || newHead.y < 0 || newHead.y >= GRID) {
    aliveRef.current = false
    setAlive(false)
    return
  }

  // Self collision
  if (snakeRef.current.some(s => s.x === newHead.x && s.y === newHead.y)) {
    aliveRef.current = false
    setAlive(false)
    return
  }

  // Eat food
  const ate = newHead.x === foodRef.current.x && newHead.y === foodRef.current.y

  if (ate) {
    snakeRef.current = [newHead, ...snakeRef.current]
    foodRef.current = randomFood(snakeRef.current)
    scoreRef.current += 10
    setScore(scoreRef.current)
  } else {
    snakeRef.current = [newHead, ...snakeRef.current.slice(0, -1)]
  }

  // Draw
  draw(ctx)

  // Schedule next tick after delay
  setTimeout(() => {
    rafRef.current = requestAnimationFrame(gameLoop)
  }, getSpeed(scoreRef.current))
}
```

`setTimeout` inside RAF is the standard pattern for a fixed-rate game loop in the browser. Pure RAF runs at 60fps — too fast for Snake. Wrapping in `setTimeout(fn, delay)` gives you a controllable tick rate that increases as the score grows.

## Direction Queue

```typescript
function isOpposite(a: string, b: string): boolean {
  return (
    (a === 'up' && b === 'down') ||
    (a === 'down' && b === 'up') ||
    (a === 'left' && b === 'right') ||
    (a === 'right' && b === 'left')
  )
}

useEffect(() => {
  function handleKey(e: KeyboardEvent) {
    const map: Record<string, typeof dirRef.current> = {
      ArrowUp: 'up', ArrowDown: 'down',
      ArrowLeft: 'left', ArrowRight: 'right',
      w: 'up', s: 'down', a: 'left', d: 'right',
    }
    const dir = map[e.key]
    if (!dir) return
    e.preventDefault()

    // Buffer max 2 key presses — prevents double-reversal on fast input
    if (dirQueueRef.current.length < 2) {
      dirQueueRef.current.push(dir)
    }
  }
  window.addEventListener('keydown', handleKey)
  return () => window.removeEventListener('keydown', handleKey)
}, [])
```

The direction queue solves a classic Snake bug: if the player presses Right then Down in rapid succession between two ticks, both inputs should register. Without a queue, only the last keypress before the tick fires applies. With a queue of 2, the second input is buffered and applied on the next tick.

## Canvas Rendering

```typescript
function draw(ctx: CanvasRenderingContext2D) {
  const canvas = ctx.canvas

  // Clear
  ctx.fillStyle = '#0f172a'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  // Draw food
  ctx.fillStyle = '#ef4444'
  ctx.beginPath()
  ctx.arc(
    foodRef.current.x * CELL + CELL / 2,
    foodRef.current.y * CELL + CELL / 2,
    CELL / 2 - 2,
    0, Math.PI * 2
  )
  ctx.fill()

  // Draw snake
  snakeRef.current.forEach((seg, i) => {
    // Head is brighter than body
    ctx.fillStyle = i === 0 ? '#4ade80' : '#16a34a'
    ctx.fillRect(
      seg.x * CELL + 1,
      seg.y * CELL + 1,
      CELL - 2,
      CELL - 2
    )
  })
}
```

The `+ 1` / `- 2` on the rect gives each segment a 1px gap, making the grid structure visible. The head uses a lighter green than the body for a subtle highlight.

## Food Placement

Food must not spawn on the snake:

```typescript
function randomFood(snake: Segment[]): Segment {
  const occupied = new Set(snake.map(s => `${s.x},${s.y}`))
  let pos: Segment
  do {
    pos = {
      x: Math.floor(Math.random() * GRID),
      y: Math.floor(Math.random() * GRID),
    }
  } while (occupied.has(`${pos.x},${pos.y}`))
  return pos
}
```

## Speed Scaling

```typescript
function getSpeed(score: number): number {
  // Start at 150ms per tick, minimum 60ms
  return Math.max(60, 150 - Math.floor(score / 50) * 10)
}
```

Every 50 points reduces the tick delay by 10ms. The game gets progressively harder without any special level system.

## Cleanup

```typescript
useEffect(() => {
  return () => {
    cancelAnimationFrame(rafRef.current)
  }
}, [])
```

Always cancel the RAF on unmount. Without this, the loop continues running after the component is gone, causes a memory leak and "Can't perform state update on an unmounted component" warnings.

## Touch Controls

For mobile, add an on-screen D-pad that pushes into the same direction queue:

```typescript
function pushDir(dir: typeof dirRef.current) {
  if (dirQueueRef.current.length < 2) {
    dirQueueRef.current.push(dir)
  }
}

// In JSX:
<button onPointerDown={() => pushDir('up')}>▲</button>
```

`onPointerDown` instead of `onClick` fires immediately without the 300ms tap delay on mobile browsers.

## The Key Insight

The entire architecture comes down to one decision: **refs for game state, state only for React UI**. Once that's clear, the rest follows naturally. The RAF loop reads refs directly — no stale closures, no useCallback dependencies, no useReducer workarounds.

---

Play it: [Snake → ultimatetools.io](https://ultimatetools.io/tools/fun-tools/snake/)

*Part of [Ultimate Tools](https://ultimatetools.io/) — free browser-based tools and games.*