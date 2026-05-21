# Building Wordle in React — Date-Seeded Daily Mode, Two-Pass Duplicate Letter Evaluation, and Streak Persistence

Wordle looks simple — a 6×5 letter grid with colour-coded feedback. But two non-obvious requirements make it interesting to build correctly: **duplicate letter evaluation** and **daily mode with no backend**.

Here's how we built the [Wordle game on Ultimate Tools](https://ultimatetools.io/tools/fun-tools/wordle/).

---

## State Shape

```tsx
type TileState = 'correct' | 'present' | 'absent' | 'empty' | 'active';
type GameMode = 'daily' | 'unlimited';
type Status = 'playing' | 'won' | 'lost';

const [guesses, setGuesses] = useState<string[]>([]);           // committed guesses
const [evaluations, setEvaluations] = useState<TileState[][]>([]);
const [currentGuess, setCurrentGuess] = useState('');
const [targetWord, setTargetWord] = useState('');
const [mode, setMode] = useState<GameMode>('daily');
const [status, setStatus] = useState<Status>('playing');
const [streak, setStreak] = useState(0);
```

---

## Daily Word — No Backend Required

The daily word is derived entirely from the date. No API call, no server, no synchronisation needed:

```tsx
function getDailyWord(wordList: string[]): string {
    const epoch = new Date('2026-01-01').getTime();
    const today = new Date();
    today.setHours(0, 0, 0, 0);
    const dayIndex = Math.floor((today.getTime() - epoch) / 86_400_000);
    return wordList[dayIndex % wordList.length];
}
```

Every user on the same calendar day computes the same `dayIndex` and lands on the same word. The word list is bundled in the component — about 2,300 common 5-letter English words as a static array.

---

## Two-Pass Duplicate Letter Evaluation

The naive approach — mark green if correct position, yellow if present anywhere — gives wrong results when the guess or target contains duplicate letters.

**Example:** Target = `CRANE`, Guess = `ARRAY`

- The `A` in position 2 of `ARRAY` is correct (position 2 of `CRANE` is also `A`) → **green**
- The `R` in position 0 of `ARRAY` is present in `CRANE` → **yellow**
- The second `A` in position 4 — `CRANE` only has one `A`, already used by the green → **grey**

A single-pass scan can't handle this. Two passes can:

```tsx
function evaluateGuess(guess: string, target: string): TileState[] {
    const result: TileState[] = Array(5).fill('absent');
    const targetPool = target.split('');
    const guessLetters = guess.split('');

    // Pass 1: lock in correct positions (green), consume those letters
    for (let i = 0; i < 5; i++) {
        if (guessLetters[i] === targetPool[i]) {
            result[i] = 'correct';
            targetPool[i] = '';    // consumed — not available for yellow matching
            guessLetters[i] = ''; // consumed — don't re-evaluate in pass 2
        }
    }

    // Pass 2: remaining letters — mark yellow if present in target pool
    for (let i = 0; i < 5; i++) {
        if (guessLetters[i] === '') continue;
        const idx = targetPool.indexOf(guessLetters[i]);
        if (idx !== -1) {
            result[i] = 'present';
            targetPool[idx] = ''; // consume to prevent double-yellow
        }
    }

    return result;
}
```

---

## On-Screen + Physical Keyboard

One shared `addLetter`, `deleteLetter`, `submitGuess` handler. Physical keyboard uses a `keydown` listener on `window`:

```tsx
useEffect(() => {
    const handler = (e: KeyboardEvent) => {
        if (status !== 'playing') return;
        if (e.key === 'Enter') submitGuess();
        else if (e.key === 'Backspace') deleteLetter();
        else if (/^[a-zA-Z]$/.test(e.key)) addLetter(e.key.toUpperCase());
    };
    window.addEventListener('keydown', handler);
    return () => window.removeEventListener('keydown', handler);
}, [status, currentGuess, guesses]);
```

The on-screen keyboard calls the same three functions — no special casing.

---

## Keyboard Colour State

The keyboard tracks the best known state for each letter across all committed guesses. `correct` beats `present` beats `absent` — once a letter is confirmed green, it stays green even if it appeared yellow in an earlier guess:

```tsx
const keyStates = useMemo<Record<string, TileState>>(() => {
    const priority: Record<TileState, number> = { correct: 3, present: 2, absent: 1, empty: 0, active: 0 };
    const states: Record<string, TileState> = {};
    guesses.forEach((guess, gi) => {
        guess.split('').forEach((letter, li) => {
            const ev = evaluations[gi][li];
            if (!states[letter] || priority[ev] > priority[states[letter]]) {
                states[letter] = ev;
            }
        });
    });
    return states;
}, [guesses, evaluations]);
```

---

## Streak and Progress Persistence

**Daily progress** (in-progress guess state) is saved to localStorage keyed by date string. On mount, if the saved date matches today, the previous state is restored:

```tsx
const todayKey = new Date().toISOString().slice(0, 10); // "2026-04-12"

// Save on every guess
localStorage.setItem(`wordle-daily-${todayKey}`, JSON.stringify({ guesses, evaluations, status }));

// Restore on mount
const saved = localStorage.getItem(`wordle-daily-${todayKey}`);
if (saved) { /* restore state */ }
```

**Streak** is a separate key — incremented on daily win, reset on daily loss:

```tsx
function updateStreak(won: boolean) {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    const yesterdayKey = yesterday.toISOString().slice(0, 10);
    const lastWin = localStorage.getItem('wordle-last-win-date');

    if (won) {
        const current = Number(localStorage.getItem('wordle-streak') || 0);
        const newStreak = (lastWin === yesterdayKey || lastWin === todayKey) ? current + 1 : 1;
        localStorage.setItem('wordle-streak', String(newStreak));
        localStorage.setItem('wordle-last-win-date', todayKey);
        setStreak(newStreak);
    } else {
        localStorage.setItem('wordle-streak', '0');
        setStreak(0);
    }
}
```

---

## Result

The full component is ~400 lines. The two interesting problems — date-seeded daily word and duplicate-letter evaluation — each have clean, self-contained solutions. No game library, no external word API.

[Play it live](https://ultimatetools.io/tools/fun-tools/wordle/)