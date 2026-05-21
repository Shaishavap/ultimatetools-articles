# Building a Memory Card Game in React — Flip Animation, Match Detection, and Timer

A memory card game looks simple but has several interesting problems: preventing double-flips mid-animation, implementing the lock-flip-compare-reveal cycle correctly, and making a CSS 3D flip feel smooth. Here's how the [Memory Game](https://ultimatetools.io/tools/fun-tools/memory-game/) is built.

## State Design

```typescript
type Difficulty = 'easy' | 'medium' | 'hard';

interface Card {
  id: number;
  emoji: string;
  pairId: number;
}

// Component state
const [cards, setCards]       = useState<Card[]>([]);
const [flipped, setFlipped]   = useState<number[]>([]);   // at most 2 ids
const [matched, setMatched]   = useState<Set<number>>(new Set());
const [moves, setMoves]       = useState(0);
const [elapsed, setElapsed]   = useState(0);
const [started, setStarted]   = useState(false);
const [won, setWon]           = useState(false);
const [locked, setLocked]     = useState(false);          // blocks clicks during animation
```

`locked` is the key. Without it, fast clicks during the reveal animation corrupt state.

## Card Generation and Shuffle

```typescript
const EMOJI_POOL = ['🐶','🐱','🐭','🐹','🐰','🦊','🐻','🐼',
                    '🦁','🐯','🐨','🐸','🦋','🐙','🦀','🐬',
                    '🦄','🐲','🌺','⭐','🍎','🍕','🎸','🚀'];

function buildDeck(pairs: number): Card[] {
  const emojis = EMOJI_POOL.slice(0, pairs);
  const deck: Card[] = [];
  emojis.forEach((emoji, pairId) => {
    deck.push({ id: pairId * 2,     emoji, pairId });
    deck.push({ id: pairId * 2 + 1, emoji, pairId });
  });
  return fisherYates(deck);
}

function fisherYates<T>(arr: T[]): T[] {
  const a = [...arr];
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}
```

Difficulty controls pair count: easy = 8 pairs (4×4), medium = 15 pairs (5×6), hard = 24 pairs (6×8).

## The Flip-Lock-Compare Cycle

This is the core logic. Clicking a card triggers:

```typescript
function handleFlip(id: number) {
  // Guard: locked, already flipped, or already matched
  if (locked) return;
  if (flipped.includes(id)) return;
  if (matched.has(cards.find(c => c.id === id)!.pairId)) return;

  const next = [...flipped, id];
  setFlipped(next);

  if (!started) {
    setStarted(true); // start timer on first click
  }

  if (next.length === 2) {
    setMoves(m => m + 1);
    setLocked(true); // block further clicks

    const [a, b] = next.map(fid => cards.find(c => c.id === fid)!);

    if (a.pairId === b.pairId) {
      // Match: add to matched set, clear flipped, unlock
      setTimeout(() => {
        setMatched(prev => new Set([...prev, a.pairId]));
        setFlipped([]);
        setLocked(false);
      }, 400); // brief pause to show both cards
    } else {
      // No match: flip both back after delay
      setTimeout(() => {
        setFlipped([]);
        setLocked(false);
      }, 900); // long enough to see both cards
    }
  }
}
```

The timeout durations matter. Too short and the player can't see what the second card was. Too long and the game feels sluggish.

## CSS 3D Flip Animation

Each card is a container with two faces — front (face-down) and back (face-up):

```tsx
<div
  className={`card-container ${isFlipped || isMatched ? 'flipped' : ''}`}
  onClick={() => handleFlip(card.id)}
>
  <div className="card-inner">
    <div className="card-front">?</div>
    <div className="card-back">{card.emoji}</div>
  </div>
</div>
```

```css
.card-container {
  perspective: 600px;
  cursor: pointer;
}

.card-inner {
  position: relative;
  width: 100%;
  height: 100%;
  transform-style: preserve-3d;
  transition: transform 0.35s ease;
}

.card-container.flipped .card-inner {
  transform: rotateY(180deg);
}

.card-front,
.card-back {
  position: absolute;
  inset: 0;
  backface-visibility: hidden;
  border-radius: 0.75rem;
  display: flex;
  align-items: center;
  justify-content: center;
}

.card-back {
  transform: rotateY(180deg);
}
```

`backface-visibility: hidden` is critical — without it, you'd see the front card showing through the back during rotation.

## Timer

Timer starts on first click, stops when all pairs are matched:

```typescript
const timerRef = useRef<ReturnType<typeof setInterval> | null>(null);

// Start on first flip
useEffect(() => {
  if (started && !won) {
    timerRef.current = setInterval(() => {
      setElapsed(e => e + 1);
    }, 1000);
  }
  return () => {
    if (timerRef.current) clearInterval(timerRef.current);
  };
}, [started, won]);

// Check win condition after each match
useEffect(() => {
  const totalPairs = cards.length / 2;
  if (matched.size === totalPairs && totalPairs > 0) {
    setWon(true);
    if (timerRef.current) clearInterval(timerRef.current);
  }
}, [matched, cards.length]);
```

## Match Animation

When a pair matches, a brief "bounce" animation fires on those two cards. This uses a `bouncing` state — a list of card IDs currently playing the bounce:

```typescript
const [bouncing, setBouncing] = useState<number[]>([]);

// Inside the match branch of handleFlip:
setTimeout(() => {
  setBouncing([a.id, b.id]);
  setTimeout(() => setBouncing([]), 400);
  setMatched(prev => new Set([...prev, a.pairId]));
  setFlipped([]);
  setLocked(false);
}, 300);
```

```css
.card-bounce {
  animation: bounce 0.4s ease;
}

@keyframes bounce {
  0%   { transform: scale(1); }
  40%  { transform: scale(1.15); }
  70%  { transform: scale(0.95); }
  100% { transform: scale(1); }
}
```

## Mismatch Shake

Similarly, a shake animation fires when two cards don't match:

```typescript
const [shaking, setShaking] = useState<number[]>([]);

// Inside the no-match branch:
setTimeout(() => {
  setShaking(next);
  setTimeout(() => setShaking([]), 500);
  setFlipped([]);
  setLocked(false);
}, 600);
```

```css
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  20%       { transform: translateX(-6px); }
  40%       { transform: translateX(6px); }
  60%       { transform: translateX(-4px); }
  80%       { transform: translateX(4px); }
}
```

## New Game

Reset everything and generate a fresh shuffled deck:

```typescript
function newGame(diff = difficulty) {
  setDifficulty(diff);
  const pairs = diff === 'easy' ? 8 : diff === 'medium' ? 15 : 24;
  setCards(buildDeck(pairs));
  setFlipped([]);
  setMatched(new Set());
  setMoves(0);
  setElapsed(0);
  setStarted(false);
  setWon(false);
  setLocked(false);
  setGeneration(g => g + 1); // triggers re-animation of card deal
}
```

The `generation` counter forces React to remount cards so the deal animation plays again. Without it, switching difficulty and pressing New Game would show the old cards briefly before swapping.

## Putting It Together

The full game runs with ~150 lines of component logic and ~60 lines of CSS. No game library, no external animation dependency. The trickiest part isn't the flip — it's the lock/unlock timing ensuring that rapid taps never leave the game in an inconsistent state.

Try it: [Memory Game → ultimatetools.io](https://ultimatetools.io/tools/fun-tools/memory-game/)