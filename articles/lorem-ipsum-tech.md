# Building a Lorem Ipsum Generator in React — No Library, Custom Word Pool, Paragraphs/Sentences/Words

Most Lorem Ipsum generators use an npm package that's overkill for what is essentially random word sampling. Here's how the [Lorem Ipsum Generator](https://ultimatetools.io/tools/text-tools/lorem-ipsum-generator/) is built — zero dependencies, three output modes, copy-to-clipboard.

## The Word Pool

Classic Lorem Ipsum comes from Cicero's "de Finibus Bonorum et Malorum". The tool uses the canonical word set:

```typescript
const WORDS = [
    "lorem", "ipsum", "dolor", "sit", "amet", "consectetur", "adipiscing", "elit",
    "sed", "do", "eiusmod", "tempor", "incididunt", "ut", "labore", "et", "dolore",
    "magna", "aliqua", "ut", "enim", "ad", "minim", "veniam", "quis", "nostrud",
    "exercitation", "ullamco", "laboris", "nisi", "ut", "aliquip", "ex", "ea",
    "commodo", "consequat", "duis", "aute", "irure", "dolor", "in", "reprehenderit",
    "in", "voluptate", "velit", "esse", "cillum", "dolore", "eu", "fugiat", "nulla",
    "pariatur", "excepteur", "sint", "occaecat", "cupidatat", "non", "proident",
    "sunt", "in", "culpa", "qui", "officia", "deserunt", "mollit", "anim", "id",
    "est", "laborum"
];
```

69 words. Repeated sampling with `Math.random()` produces varied output without exhausting the pool.

## State

```typescript
type OutputType = "paragraphs" | "sentences" | "words";

const [count, setCount]   = useState(3);
const [type, setType]     = useState<OutputType>("paragraphs");
const [text, setText]     = useState("");
const [copied, setCopied] = useState(false);
```

`count` is the number of units to generate (3 paragraphs, 5 sentences, 100 words, etc.).

## Sentence Generation

A sentence is 5–14 random words, first word capitalized, ending with a period:

```typescript
function generateSentence(): string {
    const length = Math.floor(Math.random() * 10) + 5; // 5–14 words
    let sentence = "";
    for (let i = 0; i < length; i++) {
        const word = WORDS[Math.floor(Math.random() * WORDS.length)];
        const isFirst = i === 0;
        const isLast  = i === length - 1;
        sentence += (isFirst
            ? word.charAt(0).toUpperCase() + word.slice(1)
            : word
        ) + (isLast ? "." : " ");
    }
    return sentence;
}
```

Result: `"Lorem ipsum dolor sit amet consectetur adipiscing elit sed do."`

## Paragraph Generation

A paragraph is 3–7 sentences joined with spaces:

```typescript
function generateParagraph(): string {
    const sentenceCount = Math.floor(Math.random() * 5) + 3; // 3–7 sentences
    let paragraph = "";
    for (let i = 0; i < sentenceCount; i++) {
        paragraph += generateSentence() + " ";
    }
    return paragraph.trim();
}
```

## The Generate Function

Three modes, one function:

```typescript
function generate() {
    let result = "";

    if (type === "words") {
        const words: string[] = [];
        for (let i = 0; i < count; i++) {
            words.push(WORDS[Math.floor(Math.random() * WORDS.length)]);
        }
        result = words.join(" ");

    } else if (type === "sentences") {
        for (let i = 0; i < count; i++) {
            result += generateSentence() + " ";
        }

    } else { // paragraphs
        for (let i = 0; i < count; i++) {
            result += generateParagraph() + "\n\n";
        }
    }

    setText(result.trim());
    setCopied(false);
}
```

Paragraphs are separated with `\n\n` — when displayed in a `<textarea>` or `<pre>`, this renders as blank lines between them.

## Auto-Generate on Mount

Generate text immediately when the component first renders so users see output without clicking anything:

```typescript
// Generate initial text — runs once on mount
if (!text) generate();
```

This is intentionally simple (not inside a `useEffect`) since `generate` is a pure function with no side effects and we only need it once. An alternative is:

```typescript
useEffect(() => {
    generate();
// eslint-disable-next-line react-hooks/exhaustive-deps
}, []); // intentional empty deps — generate once on mount
```

## Regenerate on Setting Change

When the user changes `type` or `count`, regenerate automatically:

```typescript
useEffect(() => {
    generate();
}, [type, count]);
```

No "Generate" button required — the output updates as you adjust the sliders.

## Copy to Clipboard

```typescript
async function copyToClipboard() {
    try {
        await navigator.clipboard.writeText(text);
    } catch {
        // Fallback for older browsers
        const el = document.createElement("textarea");
        el.value = text;
        document.body.appendChild(el);
        el.select();
        document.execCommand("copy");
        document.body.removeChild(el);
    }
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
}
```

Button toggles between "Copy" and "Copied!" for 2 seconds.

## The UI

```tsx
return (
    <div className="mx-auto max-w-4xl space-y-8">

        {/* Controls */}
        <div className="flex flex-col gap-6 sm:flex-row sm:items-end">

            {/* Output type */}
            <div className="space-y-2">
                <label className="text-sm font-medium text-zinc-500">Type</label>
                <div className="flex gap-2">
                    {(["paragraphs", "sentences", "words"] as OutputType[]).map(t => (
                        <button
                            key={t}
                            onClick={() => setType(t)}
                            className={`px-4 py-2 rounded-lg text-sm font-medium capitalize transition-colors ${
                                type === t
                                    ? "bg-violet-600 text-white"
                                    : "bg-zinc-100 dark:bg-zinc-800 text-zinc-600 dark:text-zinc-400"
                            }`}
                        >
                            {t}
                        </button>
                    ))}
                </div>
            </div>

            {/* Count slider */}
            <div className="space-y-2 flex-1">
                <label className="text-sm font-medium text-zinc-500">
                    Count: {count}
                </label>
                <input
                    type="range"
                    min={1}
                    max={type === "words" ? 200 : type === "sentences" ? 20 : 10}
                    value={count}
                    onChange={e => setCount(Number(e.target.value))}
                    className="w-full"
                />
            </div>

            {/* Regenerate button */}
            <button onClick={generate} className="flex items-center gap-2 ...">
                <RefreshCw className="h-4 w-4" /> Regenerate
            </button>
        </div>

        {/* Output */}
        <div className="relative">
            <textarea
                value={text}
                readOnly
                rows={12}
                className="w-full font-mono text-sm ..."
            />
            <button
                onClick={copyToClipboard}
                className="absolute top-3 right-3 ..."
            >
                {copied ? <Check className="h-4 w-4" /> : <Copy className="h-4 w-4" />}
                {copied ? "Copied!" : "Copy"}
            </button>
        </div>
    </div>
);
```

## The Full Picture

The whole component is ~110 lines. No npm package, no API call, no seed-based deterministic output (which is unnecessary for placeholder text). Random sampling from a 69-word pool is sufficient — users want varied placeholder text, not reproducible results.

The max range for the slider scales by type: 200 words, 20 sentences, 10 paragraphs. These limits match what's practical — 10 paragraphs is already ~500 words of placeholder text.

Try it: [Lorem Ipsum Generator → ultimatetools.io](https://ultimatetools.io/tools/text-tools/lorem-ipsum-generator/)