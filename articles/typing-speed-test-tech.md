# Building a Typing Speed Test in React — WPM Calculation, Real-Time Char Diff, and Accuracy

A typing speed test has a few interesting implementation details: the WPM formula, real-time character-by-character diff rendering, and the state machine that ties idle → running → done together. Here's how the [Typing Speed Test](https://ultimatetools.io/tools/fun-tools/typing-speed-test/) is built.

## State Machine

Three statuses keep the UI deterministic:

```typescript
type Status = 'idle' | 'running' | 'done';

const [status, setStatus]   = useState<Status>('idle');
const [input, setInput]     = useState('');
const [elapsed, setElapsed] = useState(0);   // seconds
const [wpm, setWpm]         = useState(0);
const [accuracy, setAccuracy] = useState(100);
const [passageIndex, setPassageIndex] = useState(0);
```

- `idle` — test hasn't started, show Start button
- `running` — user is typing, timer is ticking
- `done` — passage completed, show results

## Starting the Test

The test starts when the user clicks Start (not on first keypress — starting on keypress causes the first character to be typed before the state is set):

```typescript
function startGame() {
  setStatus('running');
  setInput('');
  setElapsed(0);
  setWpm(0);
  setAccuracy(100);
  // Pick a random passage (different from current one)
  setPassageIndex(i => {
    let next = Math.floor(Math.random() * PASSAGES.length);
    while (next === i) next = Math.floor(Math.random() * PASSAGES.length);
    return next;
  });
  // Focus the textarea immediately
  setTimeout(() => textareaRef.current?.focus(), 0);
}
```

## Timer

```typescript
const timerRef = useRef<ReturnType<typeof setInterval> | null>(null);

const clearTimer = useCallback(() => {
  if (timerRef.current) { clearInterval(timerRef.current); timerRef.current = null; }
}, []);

useEffect(() => {
  if (status === 'running') {
    timerRef.current = setInterval(() => setElapsed(e => e + 1), 1000);
  } else {
    clearTimer();
  }
  return clearTimer;
}, [status, clearTimer]);
```

## WPM Formula

The standard WPM formula divides character count by 5 (average word length) and divides by elapsed minutes:

```typescript
function calcWpm(correctChars: number, seconds: number): number {
  if (seconds === 0) return 0;
  const minutes = seconds / 60;
  const words = correctChars / 5;
  return Math.round(words / minutes);
}
```

Only **correct** characters count toward WPM. Typing wrong characters fast doesn't inflate your score.

## Accuracy

Accuracy is the ratio of correct characters to total characters typed:

```typescript
function calcAccuracy(passage: string, input: string): number {
  if (input.length === 0) return 100;
  let correct = 0;
  for (let i = 0; i < input.length; i++) {
    if (input[i] === passage[i]) correct++;
  }
  return Math.round((correct / input.length) * 100);
}
```

## Handling Input

The textarea `onChange` handler drives all state updates:

```typescript
function handleInput(e: React.ChangeEvent<HTMLTextAreaElement>) {
  if (status !== 'running') return;

  const val = e.target.value;
  // Prevent typing past passage length
  if (val.length > passage.length) return;

  setInput(val);

  const correct = [...val].filter((c, i) => c === passage[i]).length;
  setAccuracy(calcAccuracy(passage, val));
  setWpm(calcWpm(correct, elapsed));

  // Check completion
  if (val === passage) {
    clearTimer();
    setStatus('done');
    setWpm(calcWpm(correct, Math.max(elapsed, 1)));
    setAccuracy(calcAccuracy(passage, val));
  }
}
```

One subtlety: `elapsed` inside `handleInput` can be stale (closure over the initial value). Fix: compute WPM from the current elapsed value in a `useEffect` that watches `[elapsed, input]` instead, or pass elapsed via a ref:

```typescript
const elapsedRef = useRef(0);

useEffect(() => {
  elapsedRef.current = elapsed;
}, [elapsed]);

// Inside handleInput:
setWpm(calcWpm(correct, elapsedRef.current));
```

## Real-Time Character Diff

The passage is displayed character-by-character. Each character gets a Tailwind class based on whether it's been typed correctly, incorrectly, or not yet:

```typescript
function getCharState(
  passage: string,
  input: string
): Array<{ cls: string; wrong: boolean }> {
  return passage.split('').map((_, i) => {
    if (i >= input.length) {
      return { cls: 'text-zinc-400', wrong: false };     // not yet typed
    }
    if (input[i] === passage[i]) {
      return { cls: 'text-zinc-900 dark:text-zinc-100', wrong: false }; // correct
    }
    return {
      cls: 'text-red-500 bg-red-100 dark:bg-red-900/30 rounded',
      wrong: true,                                        // wrong char
    };
  });
}
```

Render:

```tsx
const charStates = getCharState(passage, input);

<div className="font-mono text-base leading-8 tracking-wide select-none">
  {passage.split('').map((char, i) => (
    <span
      key={i}
      className={charStates[i].cls}
      style={charStates[i].wrong ? { animation: 'char-wrong 0.15s ease' } : undefined}
    >
      {char}
    </span>
  ))}
</div>
```

The `char-wrong` animation is a brief red flash that fires when a character transitions from untyped to wrong:

```css
@keyframes char-wrong {
  0%   { background-color: theme('colors.red.300'); }
  100% { background-color: theme('colors.red.100'); }
}
```

## Detecting Newly Wrong Characters

The animation only fires on the instant a character becomes wrong — not on every render. Track the previous input to detect the transition:

```typescript
const prevInputRef = useRef('');

// Inside handleInput, after setting input:
const isNewlyWrong = (i: number) =>
  i === val.length - 1 &&                     // last typed character
  val[i] !== passage[i] &&                    // it's wrong
  prevInputRef.current.length < val.length;   // it was just typed (not a backspace)

prevInputRef.current = val;
```

## Results Panel

When `status === 'done'`, show the stats with a pop animation:

```tsx
{status === 'done' && (
  <div className="rounded-xl border border-green-200 bg-green-50 p-6 text-center space-y-3"
    style={{ animation: 'fade-slide-up 0.35s ease both' }}>
    <p className="text-xl font-bold text-green-700">Done!</p>
    <div className="flex justify-center gap-8">
      {[
        { value: wpm,         label: 'WPM',      delay: '0ms' },
        { value: `${accuracy}%`, label: 'Accuracy', delay: '80ms' },
        { value: `${elapsed}s`,  label: 'Time',     delay: '160ms' },
      ].map(({ value, label, delay }) => (
        <div key={label} style={{ animation: `stat-pop 0.45s ease ${delay} both` }}>
          <p className="text-3xl font-bold">{value}</p>
          <p className="text-sm text-zinc-500">{label}</p>
        </div>
      ))}
    </div>
    <button onClick={reset}>Try Again</button>
  </div>
)}
```

## The Full Flow

1. User sees a random passage + Start button (`idle`)
2. Clicks Start → textarea appears, timer starts (`running`)
3. Types the passage — char diff updates live, WPM recalculates on each keystroke
4. Types the last character matching the passage → timer stops, results appear (`done`)
5. Clicks Try Again → new passage, reset state (`idle`)

Total component size: ~200 lines. No external libraries for the typing logic — pure React state and DOM events.

Try it: [Typing Speed Test → ultimatetools.io](https://ultimatetools.io/tools/fun-tools/typing-speed-test/)