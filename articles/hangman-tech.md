# Building Hangman in React — SVG Progressive Drawing, Word Categories, and Physical Keyboard Events

Hangman is the simplest new game we built for [Ultimate Tools](https://ultimatetools.io/tools/fun-tools/hangman/) — no canvas, no animation loop, no physics. It's a straightforward DOM-based React component. The interesting parts are the SVG drawing approach and keyboard event handling.

---

## State Shape

```tsx
type Category = 'General' | 'Animals' | 'Countries' | 'Tech';
type Status = 'playing' | 'won' | 'lost';

const [secretWord, setSecretWord]       = useState('');
const [guessedLetters, setGuessedLetters] = useState<Set<string>>(new Set());
const [wrongCount, setWrongCount]       = useState(0);
const [category, setCategory]           = useState<Category>('General');
const [status, setStatus]               = useState<Status>('playing');

const MAX_WRONG = 6;
```

Derived values:

```tsx
const revealedLetters = secretWord.split('').filter(l => guessedLetters.has(l));
const isWon  = secretWord.length > 0 && revealedLetters.length === new Set(secretWord).size;
const isLost = wrongCount >= MAX_WRONG;
```

---

## Word Lists

Four categories, each an array of 24–35 words. Words are 5–9 letters, common enough to be recognisable:

```tsx
const WORDS: Record<Category, string[]> = {
    General: ['BRIDGE', 'CASTLE', 'DESERT', 'FOREST', 'GARDEN', ...],
    Animals: ['FALCON', 'JAGUAR', 'PENGUIN', 'DOLPHIN', 'CHEETAH', ...],
    Countries: ['BRAZIL', 'FRANCE', 'MEXICO', 'NORWAY', 'SWEDEN', ...],
    Tech: ['ALGORITHM', 'BROWSER', 'COMPILER', 'DATABASE', 'FUNCTION', ...],
};

function pickWord(cat: Category): string {
    const list = WORDS[cat];
    return list[Math.floor(Math.random() * list.length)];
}
```

---

## The Guess Function

```tsx
const guess = useCallback((letter: string) => {
    if (status !== 'playing') return;
    if (guessedLetters.has(letter)) return;

    setGuessedLetters(prev => new Set([...prev, letter]));

    if (!secretWord.includes(letter)) {
        setWrongCount(prev => prev + 1);
    }
}, [status, guessedLetters, secretWord]);
```

The `useCallback` dep array matters — the `keydown` listener captures `guess` in its closure, so it must be stable or re-bound when deps change.

---

## Physical Keyboard

```tsx
useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
        const letter = e.key.toUpperCase();
        if (/^[A-Z]$/.test(letter)) guess(letter);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
}, [guess]); // re-binds when guess callback changes
```

The on-screen keyboard calls the same `guess` function. Both inputs update the same state — no mode switching or separate handlers.

---

## SVG Gallows — Conditional Body Parts

The gallows (base, vertical pole, beam, rope) renders unconditionally. Body parts are gated on `wrongCount`:

```tsx
<svg viewBox="0 0 120 140" className="w-full max-w-[180px]">
    {/* Gallows — always visible */}
    <line x1="10" y1="135" x2="110" y2="135" stroke="currentColor" strokeWidth="3" strokeLinecap="round"/>
    <line x1="30" y1="135" x2="30"  y2="10"  stroke="currentColor" strokeWidth="3" strokeLinecap="round"/>
    <line x1="30" y1="10"  x2="75"  y2="10"  stroke="currentColor" strokeWidth="3" strokeLinecap="round"/>
    <line x1="75" y1="10"  x2="75"  y2="25"  stroke="currentColor" strokeWidth="3" strokeLinecap="round"/>

    {/* Body — conditional on wrongCount */}
    {wrongCount >= 1 && (
        <circle cx="75" cy="35" r="10" stroke="currentColor" strokeWidth="2.5" fill="none"/>
    )}
    {wrongCount >= 2 && (
        <line x1="75" y1="45" x2="75" y2="85" stroke="currentColor" strokeWidth="2.5" strokeLinecap="round"/>
    )}
    {wrongCount >= 3 && (
        <line x1="75" y1="55" x2="55" y2="72" stroke="currentColor" strokeWidth="2.5" strokeLinecap="round"/>
    )}
    {wrongCount >= 4 && (
        <line x1="75" y1="55" x2="95" y2="72" stroke="currentColor" strokeWidth="2.5" strokeLinecap="round"/>
    )}
    {wrongCount >= 5 && (
        <line x1="75" y1="85" x2="58" y2="108" stroke="currentColor" strokeWidth="2.5" strokeLinecap="round"/>
    )}
    {wrongCount >= 6 && (
        <line x1="75" y1="85" x2="92" y2="108" stroke="currentColor" strokeWidth="2.5" strokeLinecap="round"/>
    )}
</svg>
```

`currentColor` inherits from the parent's CSS `color` — works in light and dark mode without hardcoded colour values.

---

## Letter Grid Rendering

Each letter shows one of three states:

```tsx
{ALPHABET.map(letter => {
    const guessed = guessedLetters.has(letter);
    const correct = guessed && secretWord.includes(letter);
    const wrong   = guessed && !secretWord.includes(letter);

    return (
        <button
            key={letter}
            onClick={() => guess(letter)}
            disabled={guessed || status !== 'playing'}
            className={cn(
                'w-9 h-9 rounded font-bold text-sm transition-colors',
                correct && 'bg-green-500 text-white',
                wrong   && 'bg-zinc-200 text-zinc-400 dark:bg-zinc-800',
                !guessed && 'bg-white border border-zinc-300 hover:bg-zinc-50 dark:bg-zinc-900 dark:border-zinc-700'
            )}
        >
            {letter}
        </button>
    );
})}
```

---

## Category Switching

Switching category immediately starts a new game — same as clicking "New Game":

```tsx
function switchCategory(cat: Category) {
    setCategory(cat);
    setSecretWord(pickWord(cat));
    setGuessedLetters(new Set());
    setWrongCount(0);
    setStatus('playing');
}
```

---

## Result

The full component is ~260 lines. No canvas, no RAF loop — just React state, a keydown listener, and an inline SVG. The SVG approach keeps the drawing purely declarative: `wrongCount` determines what's rendered, React handles the updates.

[Play it live](https://ultimatetools.io/tools/fun-tools/hangman/)