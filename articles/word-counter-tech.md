# Building a Real-Time Word Counter in React — Unicode-Safe Splitting, Reading Time, and useRef for Instant Updates

A word counter seems trivial — split on spaces, count the parts. But accurate counting across Unicode text, real-time performance without re-renders on every keystroke, and reading time estimation all have implementation details worth getting right.

Here's how the [Word Counter](https://ultimatetools.io/tools/text-tools/word-counter/) at Ultimate Tools is built.

## Counting Words: The Right Split

The naive approach:

```typescript
const count = text.split(' ').length;
```

This breaks immediately on leading/trailing spaces, multiple consecutive spaces, newlines, and tabs — all common in pasted text. A string with just whitespace returns 1 instead of 0.

The correct approach uses a regex that matches word boundaries:

```typescript
const countWords = (text: string): number => {
  const trimmed = text.trim();
  if (!trimmed) return 0;
  return trimmed.split(/\s+/).length;
};
```

`\s+` matches any run of whitespace — spaces, tabs, newlines, carriage returns. Combined with `trim()`, this handles:
- Leading and trailing spaces → trimmed off first
- Multiple spaces between words → collapsed to one split boundary
- Line breaks in multiline text → treated as word separators
- Empty string → returns 0

## Counting Characters

Two character counts matter:

```typescript
const countCharsWithSpaces = (text: string): number => text.length;

const countCharsNoSpaces = (text: string): number =>
  text.replace(/\s/g, '').length;
```

`text.length` in JavaScript counts UTF-16 code units, not Unicode codepoints. For most text (Latin, Cyrillic, Arabic, CJK), this is fine. For emoji and some symbols that use surrogate pairs, `text.length` overcounts.

For truly Unicode-aware character counting:

```typescript
const countUnicodeChars = (text: string): number =>
  [...text].length; // spread iterates by codepoint, not code unit
```

The spread operator (`[...text]`) uses the string iterator, which yields Unicode codepoints — so `"😊".length === 2` but `[..."😊"].length === 1`.

The tool uses the spread approach for the "characters" count.

## Counting Sentences and Paragraphs

```typescript
const countSentences = (text: string): number => {
  const trimmed = text.trim();
  if (!trimmed) return 0;
  const matches = trimmed.match(/[^.!?]*[.!?]+/g);
  return matches ? matches.length : 1;
};

const countParagraphs = (text: string): number => {
  const trimmed = text.trim();
  if (!trimmed) return 0;
  return trimmed.split(/\n{2,}/).filter(p => p.trim().length > 0).length;
};
```

Sentence counting is imprecise by nature — abbreviations like "Dr." and "U.S." are common false positives. The regex `/[^.!?]*[.!?]+/g` is good enough for practical use without trying to solve a hard NLP problem.

Paragraphs are split on two or more consecutive newlines (blank lines between paragraphs), filtering empty results.

## Reading Time Estimation

```typescript
const WORDS_PER_MINUTE = 200; // average adult reading speed for screen content

const readingTime = (wordCount: number): string => {
  if (wordCount === 0) return '0 sec';
  const minutes = wordCount / WORDS_PER_MINUTE;
  if (minutes < 1) return `${Math.round(minutes * 60)} sec`;
  return `${Math.ceil(minutes)} min read`;
};
```

200 WPM is the standard benchmark for on-screen reading (silent reading of digital text). Print reading averages ~250–300 WPM, but screens are slightly slower.

For under a minute, it shows seconds (`"30 sec"`). For a minute or more, it rounds up to whole minutes (`"3 min read"`).

## Real-Time Updates: useRef Over State

Storing the text in `useState` and computing all metrics inside `useMemo` works, but on long texts with very fast typing, the memo dependencies re-evaluate on every render — which is every keystroke.

A faster approach: store the text in a ref, compute metrics in the `onChange` handler, and only put computed results in state:

```typescript
type Metrics = {
  words: number;
  chars: number;
  charsNoSpaces: number;
  sentences: number;
  paragraphs: number;
  readingTime: string;
};

const textRef = useRef('');
const [metrics, setMetrics] = useState<Metrics>(emptyMetrics);

const handleChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
  const text = e.target.value;
  textRef.current = text;

  setMetrics({
    words: countWords(text),
    chars: [...text].length,
    charsNoSpaces: [...text.replace(/\s/g, '')].length,
    sentences: countSentences(text),
    paragraphs: countParagraphs(text),
    readingTime: readingTime(countWords(text)),
  });
};
```

The `textarea` is an uncontrolled input (value driven by the DOM, not React state). React only re-renders the metrics display — not the textarea itself — on each keystroke. This avoids the cursor-jump issue that sometimes occurs with controlled textareas during rapid typing.

## Debouncing for Very Long Text

For very large pastes (10,000+ words), even the regex operations can take a few milliseconds. A debounce prevents the metrics from freezing the UI mid-paste:

```typescript
import { useDeferredValue } from 'react';

const deferredText = useDeferredValue(text);

// Compute metrics from deferredText — React schedules this
// as a low-priority update, keeping the textarea responsive
```

`useDeferredValue` (React 18) is cleaner than `setTimeout`-based debounce — React manages the scheduling and cancellation automatically.

## The Full Metrics Component

```tsx
const WordCounter = () => {
  const [text, setText] = useState('');
  const deferredText = useDeferredValue(text);

  const metrics = useMemo(() => ({
    words: countWords(deferredText),
    chars: [...deferredText].length,
    charsNoSpaces: [...deferredText.replace(/\s/g, '')].length,
    sentences: countSentences(deferredText),
    paragraphs: countParagraphs(deferredText),
    readingTime: readingTime(countWords(deferredText)),
  }), [deferredText]);

  return (
    <>
      <textarea value={text} onChange={e => setText(e.target.value)} />
      <div className="metrics">
        <span>{metrics.words} words</span>
        <span>{metrics.chars} characters</span>
        <span>{metrics.charsNoSpaces} (no spaces)</span>
        <span>{metrics.sentences} sentences</span>
        <span>{metrics.paragraphs} paragraphs</span>
        <span>{metrics.readingTime}</span>
      </div>
    </>
  );
};
```

## Key Gotchas

| Problem | Solution |
|---|---|
| Empty string returns word count 1 | `trim()` first, return 0 if empty |
| Emoji overcounts with `.length` | Use `[...text].length` (spread = codepoints) |
| Multiple spaces inflate word count | Split on `\s+` not `' '` |
| Textarea cursor jumps on controlled input | Use `useDeferredValue` or uncontrolled |
| Abbreviations ("Dr.", "U.S.") break sentence count | Accept imprecision — this is not an NLP problem |

The full tool is live at the [Word Counter](https://ultimatetools.io/tools/text-tools/word-counter/) — paste any text, get instant word, character, sentence, paragraph, and reading time counts.