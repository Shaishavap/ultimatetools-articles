# Building Flappy Bird in React — Canvas + RAF Physics, Gravity, Pipe Generation, and Speed Progression

Flappy Bird is a physics-based one-button game. That puts it in the Canvas + requestAnimationFrame category — the same pattern as Snake and Brick Breaker. All mutable game state lives in refs, not React state, to avoid stale closures in the animation loop.

Here's how we built the [Flappy Bird game on Ultimate Tools](https://ultimatetools.io/tools/fun-tools/flappy-bird/).

---

## Why Refs, Not State

The RAF callback captures its closure at creation time. If `birdY` is a React state variable, the callback always sees the value from when it was created — the stale closure problem.

The fix: put all mutable game values in refs. React state is only used for what needs to trigger a re-render: `status`, `score`, and `best`.

```tsx
const birdYRef   = useRef(250);
const birdVelRef = useRef(0);
const pipesRef   = useRef<Pipe[]>([]);
const scoreRef   = useRef(0);
const speedRef   = useRef(BASE_SPEED);
const nextPipeRef = useRef(PIPE_INTVL);
const lastTsRef  = useRef(0);
const statusRef  = useRef<Status>('idle');
const rafRef     = useRef(0);

// React state — only for rendering
const [score, setScore] = useState(0);
const [status, setStatus] = useState<Status>('idle');
const [best, setBest] = useState(() => Number(localStorage.getItem('flappy-best') || 0));
```

---

## Physics Constants

```tsx
const W = 360, H = 500, GROUND_H = 60
const BIRD_X = 80, BIRD_R = 14         // bird fixed X, radius
const GRAVITY = 1400                    // px/s²
const JUMP_VEL = -460                  // px/s, negative = upward
const PIPE_W = 58, PIPE_GAP = 155      // pipe width, gap between top/bottom
const PIPE_INTVL = 1.55                // seconds between pipe spawns
const BASE_SPEED = 210                 // initial pipe scroll speed px/s
const SPEED_INC = 18                   // speed added every 5 pipes cleared
```

These were tuned empirically — start with textbook values and adjust until the game feels fair but challenging.

---

## The RAF Loop

```tsx
function loop(ts: number) {
    const dt = Math.min((ts - lastTsRef.current) / 1000, 0.05); // cap delta at 50ms
    lastTsRef.current = ts;

    // Bird physics
    birdVelRef.current += GRAVITY * dt;
    birdYRef.current   += birdVelRef.current * dt;

    // Move + cull pipes
    pipesRef.current = pipesRef.current
        .map(p => ({ ...p, x: p.x - speedRef.current * dt }))
        .filter(p => p.x + PIPE_W > -10);

    // Spawn new pipe
    nextPipeRef.current -= dt;
    if (nextPipeRef.current <= 0) {
        nextPipeRef.current = PIPE_INTVL;
        const minGapY = 90;
        const maxGapY = H - GROUND_H - PIPE_GAP - 90;
        const gapY = minGapY + Math.random() * (maxGapY - minGapY);
        pipesRef.current.push({ x: W + 10, gapY, scored: false });
    }

    // Score + speed progression
    for (const p of pipesRef.current) {
        if (!p.scored && p.x + PIPE_W < BIRD_X) {
            p.scored = true;
            scoreRef.current += 1;
            setScore(scoreRef.current);                        // trigger React render
            if (scoreRef.current % 5 === 0) {
                speedRef.current = Math.min(
                    BASE_SPEED + SPEED_INC * 6,               // cap at 6 increments
                    speedRef.current + SPEED_INC
                );
            }
        }
    }

    // Collision detection
    if (checkCollision()) {
        handleDeath();
        return;
    }

    draw();
    rafRef.current = requestAnimationFrame(loop);
}
```

The `dt` cap at `0.05` (50ms) prevents the game from warping when the tab is backgrounded and the RAF fires after a long pause.

---

## Collision Detection

Bird is a circle (center `BIRD_X, birdY`, radius `BIRD_R`). Pipes are rectangles. We use a 5px inset on the hitbox for a forgiving feel:

```tsx
function checkCollision(): boolean {
    const bY = birdYRef.current;

    // Ground and ceiling
    if (bY + BIRD_R >= H - GROUND_H) return true;
    if (bY - BIRD_R <= 0) return true;

    for (const p of pipesRef.current) {
        const inXRange = BIRD_X + BIRD_R - 5 > p.x + 5 && BIRD_X - BIRD_R + 5 < p.x + PIPE_W - 5;
        if (!inXRange) continue;
        // Top pipe: from 0 to gapY
        if (bY - BIRD_R + 5 < p.gapY) return true;
        // Bottom pipe: from gapY + PIPE_GAP to canvas bottom
        if (bY + BIRD_R - 5 > p.gapY + PIPE_GAP) return true;
    }
    return false;
}
```

---

## Bird Rotation

The bird rotates based on vertical velocity — tilts up when rising, nose-down when falling:

```tsx
const angle = Math.max(-0.45, Math.min(0.9, birdVelRef.current / 850)) * Math.PI;
ctx.save();
ctx.translate(BIRD_X, birdYRef.current);
ctx.rotate(angle);
// draw bird centered at 0,0
ctx.restore();
```

Clamping to `[-0.45, 0.9]` radians prevents extreme angles that look unrealistic.

---

## Input — Space, Click, and Touch

All three inputs call the same `flap()` function:

```tsx
function flap() {
    if (statusRef.current === 'idle') startGame();
    if (statusRef.current !== 'playing') return;
    birdVelRef.current = JUMP_VEL;
}

// Keyboard
window.addEventListener('keydown', e => {
    if (['Space', 'KeyW', 'ArrowUp'].includes(e.code)) flap();
});

// Canvas click
canvasRef.current?.addEventListener('click', flap);

// Touch
canvasRef.current?.addEventListener('touchstart', e => { e.preventDefault(); flap(); });
```

---

## Result

The full component is ~280 lines — Canvas 2D for rendering, RAF loop with delta time for physics, all mutable state in refs. No game engine, no physics library.

[Play it live](https://ultimatetools.io/tools/fun-tools/flappy-bird/)