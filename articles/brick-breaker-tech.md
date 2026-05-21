# Building Brick Breaker in React — Ball Physics, AABB Collision, and Power-Ups on Canvas

Brick Breaker is the canonical "how do you handle collision?" game. The ball hits a brick, the brick disappears, the ball bounces — and if you get the collision response wrong, the ball clips through bricks or bounces the wrong direction.

Here's how I built [Brick Breaker](https://ultimatetools.io/tools/fun-tools/brick-breaker/) with a proper AABB collision response, canvas rendering, and a power-up drop system.

## Game Loop Architecture

Same pattern as Snake: all mutable state in refs, RAF + setTimeout for a fixed tick rate.

```typescript
const stateRef = useRef<GameState>(initState())
const rafRef = useRef<number>(0)
const canvasRef = useRef<HTMLCanvasElement>(null)

function gameLoop() {
  const state = stateRef.current
  if (!state.alive) return

  update(state)
  draw(state, canvasRef.current!.getContext('2d')!)

  rafRef.current = requestAnimationFrame(gameLoop)
}
```

`update()` advances physics. `draw()` renders the current frame. Both operate on the same ref — no state copies, no diffing.

## Ball Physics

The ball has a position and a velocity vector:

```typescript
interface Ball {
  x: number
  y: number
  vx: number  // pixels per frame
  vy: number
  radius: number
}
```

Each frame, advance position by velocity:

```typescript
function moveBall(ball: Ball, canvas: HTMLCanvasElement) {
  ball.x += ball.vx
  ball.y += ball.vy

  // Wall bounce (left/right)
  if (ball.x - ball.radius <= 0) {
    ball.x = ball.radius
    ball.vx = Math.abs(ball.vx)
  }
  if (ball.x + ball.radius >= canvas.width) {
    ball.x = canvas.width - ball.radius
    ball.vx = -Math.abs(ball.vx)
  }

  // Ceiling bounce
  if (ball.y - ball.radius <= 0) {
    ball.y = ball.radius
    ball.vy = Math.abs(ball.vy)
  }
}
```

Using `Math.abs()` instead of negating ensures the velocity always goes the right direction after a bounce, even if the ball slightly penetrates the wall before the collision is detected.

## Paddle Collision with Angle Control

When the ball hits the paddle, the bounce angle depends on *where* it hits — center gives a steep return, edges give a shallow angle. This is what makes Brick Breaker strategic rather than just reactive.

```typescript
function handlePaddleCollision(ball: Ball, paddle: Paddle) {
  if (
    ball.y + ball.radius >= paddle.y &&
    ball.y - ball.radius <= paddle.y + paddle.height &&
    ball.x >= paddle.x &&
    ball.x <= paddle.x + paddle.width &&
    ball.vy > 0 // only when ball moving downward
  ) {
    const hitPos = (ball.x - paddle.x) / paddle.width // 0.0 to 1.0
    const angle = (hitPos - 0.5) * (Math.PI * 0.75) // -67.5° to +67.5°

    const speed = Math.sqrt(ball.vx ** 2 + ball.vy ** 2)
    ball.vx = speed * Math.sin(angle)
    ball.vy = -speed * Math.cos(angle) // always up after paddle hit

    ball.y = paddle.y - ball.radius // push out of paddle
  }
}
```

The angle formula maps the hit position to a range of ±67.5°. Dead center returns the ball straight up. Far edges return it at a shallow angle. The speed magnitude is preserved — only the direction changes.

## AABB Brick Collision

Each brick is an axis-aligned bounding box (AABB). When the ball hits a brick, determine which face was hit and reflect the appropriate velocity component.

```typescript
interface Brick {
  x: number
  y: number
  width: number
  height: number
  hp: number       // hits remaining
  color: string
  powerUp?: PowerUpType
}

function checkBrickCollision(ball: Ball, brick: Brick): boolean {
  // Closest point on brick to ball center
  const closestX = Math.max(brick.x, Math.min(ball.x, brick.x + brick.width))
  const closestY = Math.max(brick.y, Math.min(ball.y, brick.y + brick.height))

  const dx = ball.x - closestX
  const dy = ball.y - closestY

  if (dx * dx + dy * dy > ball.radius * ball.radius) return false

  // Determine dominant collision axis
  const overlapX = (brick.width / 2) - Math.abs(ball.x - (brick.x + brick.width / 2))
  const overlapY = (brick.height / 2) - Math.abs(ball.y - (brick.y + brick.height / 2))

  if (overlapX < overlapY) {
    ball.vx *= -1
    ball.x += ball.vx > 0 ? overlapX : -overlapX
  } else {
    ball.vy *= -1
    ball.y += ball.vy > 0 ? overlapY : -overlapY
  }

  return true
}
```

The "closest point on AABB to circle center" test is a reliable broad-phase check. The overlap comparison for determining which axis to reflect on handles corner hits gracefully — instead of clipping through, the ball bounces off the dominant face.

## Processing All Bricks Per Frame

Iterate all bricks each frame, handle collision, reduce HP, drop power-up if destroyed:

```typescript
function updateBricks(state: GameState) {
  let destroyed = 0

  for (let i = state.bricks.length - 1; i >= 0; i--) {
    const brick = state.bricks[i]
    if (!checkBrickCollision(state.ball, brick)) continue

    brick.hp -= 1
    if (brick.hp <= 0) {
      // Drop power-up with 20% probability
      if (brick.powerUp || Math.random() < 0.2) {
        state.powerUps.push(spawnPowerUp(brick))
      }
      state.bricks.splice(i, 1)
      state.score += 10
      destroyed++
    }

    // Only handle one brick collision per frame to prevent tunneling
    break
  }

  if (state.bricks.length === 0) {
    nextLevel(state)
  }
}
```

The `break` after the first collision prevents the ball from "eating" multiple bricks in a single frame when it's at a corner between two bricks. One collision per frame, then re-evaluate next frame.

## Power-Up System

Power-ups fall from destroyed bricks as collectible items:

```typescript
type PowerUpType = 'wide-paddle' | 'multi-ball' | 'slow-ball' | 'laser'

interface PowerUp {
  x: number
  y: number
  type: PowerUpType
  vy: number  // falling speed
}

function updatePowerUps(state: GameState) {
  for (let i = state.powerUps.length - 1; i >= 0; i--) {
    const pu = state.powerUps[i]
    pu.y += pu.vy

    // Check paddle catch
    if (
      pu.y + 10 >= state.paddle.y &&
      pu.x >= state.paddle.x &&
      pu.x <= state.paddle.x + state.paddle.width
    ) {
      applyPowerUp(state, pu.type)
      state.powerUps.splice(i, 1)
      continue
    }

    // Off screen — remove
    if (pu.y > state.canvas.height) {
      state.powerUps.splice(i, 1)
    }
  }
}

function applyPowerUp(state: GameState, type: PowerUpType) {
  switch (type) {
    case 'wide-paddle':
      state.paddle.width = Math.min(state.paddle.width * 1.5, 200)
      setTimeout(() => state.paddle.width = PADDLE_DEFAULT_WIDTH, 8000)
      break
    case 'slow-ball':
      const speed = Math.sqrt(state.ball.vx ** 2 + state.ball.vy ** 2)
      const factor = 0.6
      state.ball.vx *= factor
      state.ball.vy *= factor
      setTimeout(() => {
        // Restore speed ratio, not direction
        const cur = Math.sqrt(state.ball.vx ** 2 + state.ball.vy ** 2)
        if (cur > 0) {
          state.ball.vx = (state.ball.vx / cur) * speed
          state.ball.vy = (state.ball.vy / cur) * speed
        }
      }, 8000)
      break
    case 'multi-ball':
      spawnExtraBalls(state)
      break
  }
}
```

Power-up timers use `setTimeout` that writes directly to the ref state. Since we're not using React state for game objects, this is fine — the game loop reads the ref every frame regardless.

## Paddle Mouse + Touch Tracking

```typescript
useEffect(() => {
  const canvas = canvasRef.current!

  function movePaddle(clientX: number) {
    const rect = canvas.getBoundingClientRect()
    const x = clientX - rect.left - stateRef.current.paddle.width / 2
    stateRef.current.paddle.x = Math.max(
      0,
      Math.min(x, canvas.width - stateRef.current.paddle.width)
    )
  }

  canvas.addEventListener('mousemove', e => movePaddle(e.clientX))
  canvas.addEventListener('touchmove', e => {
    e.preventDefault()
    movePaddle(e.touches[0].clientX)
  }, { passive: false })

  return () => {
    canvas.removeEventListener('mousemove', movePaddle)
    canvas.removeEventListener('touchmove', movePaddle)
  }
}, [])
```

`{ passive: false }` on touchmove allows `e.preventDefault()`, which stops the page from scrolling while playing.

## Level Progression

When all bricks are cleared, advance to the next level — increase ball speed, add a new brick row:

```typescript
function nextLevel(state: GameState) {
  state.level += 1
  state.bricks = buildBricks(state.level)

  // Speed increases by 10% per level, capped at 2×
  const speedMultiplier = Math.min(1 + (state.level - 1) * 0.1, 2.0)
  const baseSpeed = BASE_SPEED * speedMultiplier
  const angle = Math.atan2(state.ball.vy, state.ball.vx)
  state.ball.vx = baseSpeed * Math.cos(angle)
  state.ball.vy = baseSpeed * Math.sin(angle)

  // Reset ball to paddle
  state.ball.x = state.canvas.width / 2
  state.ball.y = state.paddle.y - state.ball.radius - 2
  state.ball.vy = -Math.abs(state.ball.vy)
}
```

Preserving the angle while scaling the speed means the ball direction stays the same but gets faster — rather than jolting to a fixed direction.

---

Play it: [Brick Breaker → ultimatetools.io](https://ultimatetools.io/tools/fun-tools/brick-breaker/)

*Part of [Ultimate Tools](https://ultimatetools.io/) — free browser-based tools and games.*