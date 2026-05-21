# Building Whack-a-Mole in React — Recursive setTimeout Spawning, Speed Progression, and Per-Hole Timers

Whack-a-Mole is a 60-second DOM-based game: nine holes in a 3×3 grid, moles pop up at random, click to score. The interesting technical problem is the spawn system — how to manage mole timing, overlap limits, and speed progression without fighting React's rendering.

Here's how we built the [Whack-a-Mole game on Ultimate Tools](https://ultimatetools.io/tools/fun-tools/whack-a-mole/).

---

## Why Recursive setTimeout, Not setInterval

A single `setInterval` for spawning seems obvious — but it has problems:

1. **Mole duration and spawn interval are independent.** A mole spawned at `t=0` might hide at `t=850ms`. The next spawn could fire at `t=900ms`. `setInterval` can't adapt to this.
2. **Max active moles changes over time.** You don't want 4 simultaneous moles in the first 10 seconds. The spawn decision needs to read current elapsed time — not a fixed interval.

Recursive `setTimeout` solves both:

```tsx
function scheduleNext() {
    const elapsed = elapsedRef.current;
    const delay = getSpawnDelay(elapsed);

    spawnTimerRef.current = window.setTimeout(() => {
        if (statusRef.current !== 'playing') return;
        trySpawn();
        scheduleNext(); // schedule next after this one fires
    }, delay);
}
```

Each spawn decision happens at exactly the right moment, reads current elapsed time, and schedules the next one dynamically.

---

## Speed Progression

Two functions control difficulty:

```tsx
function getSpawnDelay(elapsed: number): number {
    if (elapsed < 15) return 1400; // slow opening
    if (elapsed < 30) return 1100;
    if (elapsed < 45) return 900;
    return 720;                    // fast final stretch
}

function getMaxMoles(elapsed: number): number {
    if (elapsed < 20) return 2;
    if (elapsed < 40) return 3;
    return 4;                      // peak chaos
}
```

---

## Mole State

Active moles are tracked as a boolean array — one entry per hole:

```tsx
const [activeMoles, setActiveMoles] = useState<boolean[]>(Array(HOLES).fill(false));
const activeMolesRef = useRef<boolean[]>(Array(HOLES).fill(false)); // ref for sync access in timers
```

Both state and ref are kept in sync. State drives React rendering. The ref is read inside setTimeout callbacks where the React state would be stale.

---

## Per-Hole Timers

Each of the 9 holes has its own hide-timer:

```tsx
const moleTimers = useRef<(ReturnType<typeof setTimeout> | null)[]>(
    Array(HOLES).fill(null)
);

function activateMole(hole: number) {
    // Update both ref and state
    activeMolesRef.current[hole] = true;
    setActiveMoles(prev => {
        const next = [...prev];
        next[hole] = true;
        return next;
    });

    // Auto-hide after random duration
    const duration = MOLE_MIN_MS + Math.random() * (MOLE_MAX_MS - MOLE_MIN_MS);
    moleTimers.current[hole] = setTimeout(() => {
        deactivateMole(hole);
    }, duration);
}

function deactivateMole(hole: number) {
    if (moleTimers.current[hole]) {
        clearTimeout(moleTimers.current[hole]!);
        moleTimers.current[hole] = null;
    }
    activeMolesRef.current[hole] = false;
    setActiveMoles(prev => {
        const next = [...prev];
        next[hole] = false;
        return next;
    });
}
```

---

## Spawning Logic

`trySpawn` picks a random inactive hole, checks the active mole count against the current cap, and activates if conditions are met:

```tsx
function trySpawn() {
    const elapsed = elapsedRef.current;
    const maxMoles = getMaxMoles(elapsed);
    const currentActive = activeMolesRef.current.filter(Boolean).length;

    if (currentActive >= maxMoles) return; // at cap — skip this spawn

    // Pick a random inactive hole
    const inactive = activeMolesRef.current
        .map((active, i) => (!active ? i : -1))
        .filter(i => i !== -1);

    if (inactive.length === 0) return;

    const hole = inactive[Math.floor(Math.random() * inactive.length)];
    activateMole(hole);
}
```

---

## The Game Timer

The 60-second countdown runs in a `setInterval` updated every 100ms for smooth display. Elapsed time is tracked in a ref (not state) to avoid re-creating the spawn timer when it updates:

```tsx
const elapsedRef = useRef(0);
const statusRef  = useRef<Status>('idle');

const timerInterval = useRef<ReturnType<typeof setInterval> | null>(null);

function startTimer() {
    const start = Date.now();
    timerInterval.current = setInterval(() => {
        const elapsed = (Date.now() - start) / 1000;
        elapsedRef.current = elapsed;
        const remaining = Math.max(0, GAME_DURATION - elapsed);
        setTimeLeft(Math.ceil(remaining));

        if (elapsed >= GAME_DURATION) endGame();
    }, 100);
}
```

---

## Mole CSS Animation

Moles pop up and retreat using a CSS translate transform. No canvas, no JS animation — just a class toggle:

```tsx
<div
    className={cn(
        'transition-transform duration-150 ease-out',
        activeMoles[i] ? 'translate-y-0' : 'translate-y-full'
    )}
>
    {/* mole graphic */}
</div>
```

The hole container has `overflow-hidden` — when the mole is at `translate-y-full`, it's entirely below the hole boundary and invisible. When active, it slides up to `translate-y-0`.

---

## Whack Handler

Clicking a mole deactivates it immediately (clearing its auto-hide timer), increments score, and avoids double-counting via the ref check:

```tsx
function whack(hole: number) {
    if (!activeMolesRef.current[hole]) return; // already hidden
    if (statusRef.current !== 'playing') return;

    deactivateMole(hole);
    setScore(prev => prev + 1);
    scoreRef.current += 1;
}
```

---

## Cleanup

All timers cleared on unmount and on game end:

```tsx
useEffect(() => {
    return () => {
        clearInterval(timerInterval.current!);
        clearTimeout(spawnTimerRef.current!);
        moleTimers.current.forEach(t => t && clearTimeout(t));
    };
}, []);
```

---

## Result

The full component is ~270 lines. Recursive `setTimeout` handles the spawn system cleanly. Per-hole timers give each mole independent hide behavior. CSS translate handles the pop animation without canvas or JS animation libraries.

[Play it live](https://ultimatetools.io/tools/fun-tools/whack-a-mole/)