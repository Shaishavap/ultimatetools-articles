# Building 2048, Wordle, Flappy Bird, Hangman, and Whack-a-Mole in React — Architecture Notes

We recently added five more browser games to [ultimatetools.io](https://ultimatetools.io/tools/fun-tools/) — 2048, Wordle, Flappy Bird, Hangman, and Whack-a-Mole. All built in React, no game libraries, running in the browser.

Here's what was interesting or non-obvious about each one.

---

## Shared Architecture

Each game follows the same two-file pattern used across the site:

```
app/tools/fun-tools/[game]/page.tsx   ← server component: metadata + SEO schemas
components/tools/[GameName].tsx        ← "use client": all game logic
```

**State approach varies by game:**
- DOM-based games (2048, Wordle, Hangman): `useState` is fine — updates are infrequent
- Canvas + RAF games (Flappy Bird, Whack-a-Mole): all game state in `useRef` — avoids stale closures inside the animation loop

---

## 2048 — The Direction Rotation Trick

2048 is a 4×4 grid where tiles slide and merge when you swipe/press a direction.

The naive approach: write four separate merge functions for left, right, up, and down. That's a lot of duplicated logic.

**The better approach:** write one `mergeLeft` function, then rotate the grid before and after calling it.

```typescript
function rotateGrid(grid: number[][]): number[][] {
  const n = grid.length;
  return Array.from({ length: n }, (_, r) =>
    Array.from({ length: n }, (_, c) => grid[n - 1 - c][r])
  );
}

function move(grid: number[][], direction: Direction) {
  let g = grid;
  if (direction === "right") g = rotateGrid(rotateGrid(g));
  if (direction === "up")    g = rotateGrid(rotateGrid(rotateGrid(g)));
  if (direction === "down")  g = rotateGrid(g);
  
  g = g.map(mergeLeft);  // always merge left
  
  if (direction === "right") g = rotateGrid(rotateGrid(g));
  if (direction === "up")    g = rotateGrid(g);
  if (direction === "down")  g = rotateGrid(rotateGrid(rotateGrid(g)));
  return g;
}
```

`mergeLeft` slides all tiles left, merges adjacent equal tiles, and fills with zeros. One function handles all four directions.

**Slide animations:** CSS transitions on tile positions. Tiles are keyed by a stable ID (not value or position) so React doesn't remount them on each move — the key stays the same, the `transform: translate()` changes, and CSS handles the animation.

---

## Wordle — Two-Pass Duplicate Letter Evaluation

Wordle has a subtle edge case: **duplicate letters**. 

If the secret word is `ABBEY` and you guess `EERIE`:
- The first `E` should be yellow (E is in the word, wrong position)
- The second `E` should be gray (the word only has one E, already accounted for)
- The third `E` should be gray (same reason)

A single-pass evaluation gets this wrong. The correct approach is two passes:

```typescript
function evaluate(guess: string, target: string): LetterState[] {
  const result: LetterState[] = Array(5).fill("absent");
  const targetChars = target.split("");
  const guessChars = guess.split("");

  // Pass 1: mark exact matches (green)
  guessChars.forEach((char, i) => {
    if (char === targetChars[i]) {
      result[i] = "correct";
      targetChars[i] = "#"; // consume this target letter
    }
  });

  // Pass 2: mark wrong-position matches (yellow)
  guessChars.forEach((char, i) => {
    if (result[i] === "correct") return;
    const idx = targetChars.indexOf(char);
    if (idx !== -1) {
      result[i] = "present";
      targetChars[idx] = "#"; // consume so it's not matched again
    }
  });

  return result;
}
```

**Daily mode seeding:** The daily word is selected by taking the day offset from a fixed epoch and indexing into the word list. Same result for every user on the same day, no server needed.

```typescript
const EPOCH = new Date("2024-01-01").getTime();
const dayIndex = Math.floor((Date.now() - EPOCH) / 86400000) % wordList.length;
const dailyWord = wordList[dayIndex];
```

---

## Flappy Bird — Physics in a RAF Loop

Flappy Bird is Canvas + `requestAnimationFrame`. All state lives in refs to avoid stale closures:

```typescript
const birdYRef = useRef(200);
const velocityRef = useRef(0);
const pipesRef = useRef<Pipe[]>([]);
const scoreRef = useRef(0);
const statusRef = useRef<"idle" | "playing" | "dead">("idle");
```

**Physics per frame:**
```typescript
velocityRef.current += GRAVITY;           // ~0.5 per frame
birdYRef.current += velocityRef.current;
```

**Jump:**
```typescript
velocityRef.current = JUMP_VELOCITY;      // ~-9, negative = up
```

**Pipe generation:** a new pipe spawns every N frames. Pipe gap position is randomized within a constrained range so the game stays playable. Speed increases every 10 points.

**Collision detection:** axis-aligned bounding box (AABB) — simple rectangle overlap between bird bounds and each pipe rect. Checked every frame.

The bird is drawn as a rounded rectangle with a wing, not a sprite — keeps the asset count at zero.

---

## Hangman — SVG Progressive Drawing

Hangman's "drawing" is an SVG with 7 stages. Each stage is a path or element that becomes visible as wrong guesses accumulate:

```typescript
const PARTS = [
  <line x1="60" y1="20" x2="60" y2="120" />,  // body
  <circle cx="60" cy="15" r="10" />,            // head
  <line x1="60" y1="50" x2="40" y2="80" />,    // left arm
  <line x1="60" y1="50" x2="80" y2="80" />,    // right arm
  <line x1="60" y1="120" x2="40" y2="150" />,  // left leg
  <line x1="60" y1="120" x2="80" y2="150" />,  // right leg
  <line x1="20" y1="180" x2="100" y2="180" />, // ground
];

// Render parts[0..wrongCount]
```

**Keyboard handling:** both on-screen letter buttons and physical keyboard events (`keydown` listener). The physical keyboard handler checks that the key is a letter, hasn't been guessed, and the game is in progress.

**Word categories:** General, Animals, Countries, Tech — each a pre-defined array of words. Category selector shown at game start. Selecting a new category resets the game.

---

## Whack-a-Mole — Recursive setTimeout Spawning

Whack-a-Mole has 9 holes. Moles pop up randomly, stay for a fixed duration, then retract.

The approach that works well: **per-hole recursive setTimeout**, not a single game tick.

```typescript
function spawnMole() {
  const hole = pickRandomEmptyHole();
  setMoles(prev => ({ ...prev, [hole]: true }));

  setTimeout(() => {
    setMoles(prev => ({ ...prev, [hole]: false }));
    if (gameRunning) {
      setTimeout(spawnMole, getRandomDelay()); // recursive — next mole
    }
  }, MOLE_VISIBLE_MS);
}
```

**Speed progression:** `MOLE_VISIBLE_MS` decreases every 15 seconds. More moles appear faster as the game progresses.

**Hit detection:** clicking a mole calls `whack(hole)` which checks `moles[hole] === true`, increments score, and immediately sets `moles[hole] = false`. The mole visually retracts on the next render.

**60-second timer:** a single `setInterval` ticks every second, decrements time, and ends the game at 0. On game end, all active mole timeouts are effectively abandoned — they fire but the `gameRunning` check prevents new spawns.

---

## Common Patterns Across All Five

**localStorage for scores/streaks:**
```typescript
useEffect(() => {
  const saved = localStorage.getItem("best-score");
  if (saved) setBestScore(parseInt(saved));
}, []);
```

**Touch support:** all Canvas games listen for `touchstart`/`touchend` in addition to `mousedown`/`keydown`.

**ShareResultCard:** the existing share component is reused for end-of-game share images — same canvas generation pattern as every other tool on the site.

---

All five games are live at **[ultimatetools.io/tools/fun-tools/](https://ultimatetools.io/tools/fun-tools/)** — no login, no download, works on any browser.