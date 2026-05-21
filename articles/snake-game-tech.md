# Building Snake in React — Canvas RAF Loop, Mutable Refs to Avoid Stale Closures, and Wall Wrap

Snake seems simple — move, eat, grow, repeat. But building it in React has a specific trap: if you use `useState` for game state inside a `requestAnimationFrame` loop, every callback captures stale values from the render when it was created. Here's how the [Snake game](https://ultimatetools.io/tools/fun-tools/snake/) is built — with all mutable game state in refs, timestamp-based speed control, wall wrap, and touch support.

## The Stale Closure Problem

In React, `useState` values inside a RAF callback are frozen at the time the callback was created. If you write `const [snake, setSnake] = useState(...)` and read `snake` inside `requestAnimationFrame`, you'll always see the initial empty array — even after the snake grows.

The fix is to put all game state that the RAF loop needs to read into `useRef`:

```typescript
const snakeRef      = useRef<Pt[]>([]);
const dirRef        = useRef<Dir>("RIGHT");
const nextDirRef    = useRef<Dir>("RIGHT");
const foodRef       = useRef<Pt>({ x: 5, y: 5 });
const scoreRef      = useRef(0);
const speedRef      = useRef(BASE_SPEED);
const lastTickRef   = useRef(0);
const statusRef     = useRef<Status>("idle");
const rafRef        = useRef<number>(0);
```

Refs are mutable objects — `ref.current` always gives you the latest value, not a stale closure. `useState` is only used for values that need to trigger a re-render (score display, status for overlay).

## The Game Loop

Instead of `setInterval`, the game loop uses `requestAnimationFrame` with timestamp-based speed control:

```typescript
const tick = useCallback((ts: number) => {
    if (statusRef.current !== "playing") return;
    rafRef.current = requestAnimationFrame(tick);

    // Only advance the game when enough time has passed
    if (ts - lastTickRef.current < speedRef.current) {
        draw();
        return;
    }
    lastTickRef.current = ts;

    // ... move snake, check collisions
}, [draw, placeFood]);
```

`requestAnimationFrame` runs at 60fps. The speed is controlled by how often the game *logic* runs, not the draw rate. When `ts - lastTickRef < speedRef`, only the draw happens — positions don't advance. This separates render frequency from game tick frequency.

Starting with `BASE_SPEED = 150ms`, the snake moves 6–7 cells per second. `MIN_SPEED = 65ms` is the floor (about 15 cells/second).

## Wall Wrap

When the snake reaches an edge, it wraps to the opposite side:

```typescript
const next: Pt = {
    x: (head.x + (dir === "RIGHT" ? 1 : dir === "LEFT" ? -1 : 0) + COLS) % COLS,
    y: (head.y + (dir === "DOWN"  ? 1 : dir === "UP"   ? -1 : 0) + ROWS) % ROWS,
};
```

The `+ COLS` before `% COLS` handles the left edge case: `(-1 + 20) % 20 = 19`. Without it, JavaScript's modulo for negative numbers gives `-1 % 20 = -1` (not 19).

## Speed Scaling

Every 5 food eaten, the snake speeds up:

```typescript
const newScore = scoreRef.current + 10;
scoreRef.current = newScore;
setScore(newScore);

if ((newScore / 10) % 5 === 0) {
    speedRef.current = Math.max(MIN_SPEED, speedRef.current - SPEED_STEP);
}
```

`newScore / 10` converts score to food count. When food count is a multiple of 5 (5, 10, 15...), subtract `SPEED_STEP = 10ms` from the interval. Floor at `MIN_SPEED = 65ms`.

## Buffering Direction Changes

Two refs handle direction: `dirRef` (current) and `nextDirRef` (queued):

```typescript
const onKey = (e: KeyboardEvent) => {
    const d = KEY_DIR[e.key];
    if (!d) return;
    // Block 180-degree reversal
    if (d !== OPPOSITE[dirRef.current]) nextDirRef.current = d;
};
```

At the start of each tick:
```typescript
dirRef.current = nextDirRef.current;
```

The separation prevents rapid double-key presses from causing a 180-degree turn within the same tick. `OPPOSITE` is a lookup: `{ UP: "DOWN", DOWN: "UP", LEFT: "RIGHT", RIGHT: "LEFT" }`.

## Touch / Swipe

```typescript
const onTouchStart = (e: React.TouchEvent) => {
    touchStartRef.current = { x: e.touches[0].clientX, y: e.touches[0].clientY };
};

const onTouchEnd = (e: React.TouchEvent) => {
    const dx = e.changedTouches[0].clientX - touchStartRef.current.x;
    const dy = e.changedTouches[0].clientY - touchStartRef.current.y;

    if (Math.abs(dx) < 12 && Math.abs(dy) < 12) return; // ignore taps

    const d: Dir = Math.abs(dx) > Math.abs(dy)
        ? (dx > 0 ? "RIGHT" : "LEFT")
        : (dy > 0 ? "DOWN"  : "UP");

    if (d !== OPPOSITE[dirRef.current]) nextDirRef.current = d;
};
```

The 12px threshold filters out taps. The dominant axis determines the direction — horizontal swipe overrides vertical if `|dx| > |dy|`.

## Drawing

The game renders on a 400×400px canvas (20 columns × 20 rows × 20px each):

```typescript
// Background
ctx.fillStyle = "#09090b";
ctx.fillRect(0, 0, W, H);

// Subtle grid lines
ctx.strokeStyle = "#18181b";
ctx.lineWidth = 0.5;
for (let x = 0; x <= COLS; x++) {
    ctx.beginPath(); ctx.moveTo(x * CELL, 0); ctx.lineTo(x * CELL, H); ctx.stroke();
}
```

**Food** uses a radial gradient for a glow effect:

```typescript
const grd = ctx.createRadialGradient(fx, fy, 1, fx, fy, CELL);
grd.addColorStop(0, "rgba(244,63,94,0.55)");
grd.addColorStop(1, "rgba(244,63,94,0)");
ctx.fillStyle = grd;
ctx.fillRect(food.x * CELL, food.y * CELL, CELL, CELL);
```

**Snake** — the head is solid violet, the tail fades to darker purple:

```typescript
snake.forEach((seg, i) => {
    const t = i / Math.max(snake.length - 1, 1);
    const lightness = Math.round(55 - t * 22);
    ctx.fillStyle = isHead ? "#7c3aed" : `hsl(263,70%,${lightness}%)`;
    ctx.roundRect(seg.x * CELL + 1, seg.y * CELL + 1, CELL - 2, CELL - 2, isHead ? 6 : 4);
    ctx.fill();
});
```

`t = 0` is the head (lightness 55%), `t = 1` is the tail tip (lightness 33%). The gradient from purple to dark purple is purely computed — no array of colors needed.

## localStorage High Score

```typescript
useEffect(() => {
    const hs = parseInt(localStorage.getItem("snake-hs") ?? "0");
    if (hs) setHighScore(hs);
}, []);

// On game over:
const hs = parseInt(localStorage.getItem("snake-hs") ?? "0");
if (scoreRef.current > hs) {
    localStorage.setItem("snake-hs", String(scoreRef.current));
    setHighScore(scoreRef.current);
}
```

`setHighScore` triggers a re-render to update the display. Writing to `localStorage` is synchronous and doesn't cause a re-render — fine for a single key.

## The Shape of the Game State

The key design choice is keeping React state minimal. React state holds only what the UI needs to display: `score`, `highScore`, `status` (for showing overlays). Everything the RAF loop touches lives in refs. This avoids stale closures entirely and keeps the game loop free of React's render cycle.

Play it: [Snake → ultimatetools.io](https://ultimatetools.io/tools/fun-tools/snake/)