# Building a URL Encoder/Decoder with Live Mode in Next.js

URL encoding is one of those things every developer needs but nobody wants to open a terminal for. I built a browser-based URL encoder/decoder with live processing, mode switching, and zero server calls. Here's how it works under the hood.

The live tool is at [URL Encoder/Decoder](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/).

---

## Why encodeURIComponent and not encodeURI?

JavaScript gives you two encoding functions, and picking the wrong one is a common source of bugs.

**`encodeURI`** encodes a full URL but preserves structural characters like `:`, `/`, `?`, `&`, `=`, and `#`. It's meant for encoding an entire URL string where you want the structure intact.

**`encodeURIComponent`** encodes everything except letters, digits, and `- _ . ! ~ * ' ( )`. It's meant for encoding a single value — like a query parameter.

```js
encodeURI("https://example.com/search?q=hello world&lang=en")
// "https://example.com/search?q=hello%20world&lang=en"
// Structure preserved, only the space is encoded

encodeURIComponent("hello world&lang=en")
// "hello%20world%26lang%3Den"
// Everything encoded — &, = included
```

If you're encoding a query parameter value, `encodeURIComponent` is always the right choice. That's what this tool uses.

---

## The core processing function

The entire encode/decode logic fits in one function:

```tsx
const process = (text: string, currentMode: "encode" | "decode") => {
    try {
        setError("");
        if (!text) {
            setOutput("");
            return;
        }
        if (currentMode === "encode") {
            setOutput(encodeURIComponent(text));
        } else {
            setOutput(decodeURIComponent(text));
        }
    } catch (err) {
        setError("Invalid format for decoding");
    }
};
```

The `try/catch` matters. `encodeURIComponent` never throws — any string is valid input. But `decodeURIComponent` will throw a `URIError` if the input contains malformed percent sequences like `%ZZ` or a lone `%` at the end. The catch block handles this gracefully instead of crashing the UI.

---

## Live mode with useEffect

The tool processes input in real-time by default:

```tsx
const [input, setInput] = useState("");
const [output, setOutput] = useState("");
const [mode, setMode] = useState<"encode" | "decode">("encode");
const [liveMode, setLiveMode] = useState(true);

useEffect(() => {
    if (liveMode) {
        process(input, mode);
    }
}, [input, mode, liveMode]);
```

Every keystroke triggers the effect. Since `encodeURIComponent` and `decodeURIComponent` are synchronous and fast (they're native browser functions, not JavaScript implementations), there's no performance concern — even with large inputs, the encoding happens in under a millisecond.

When live mode is disabled, a manual button appears:

```tsx
{!liveMode && (
    <button onClick={() => process(input, mode)}>
        {mode === "encode" ? "Encode" : "Decode"}
    </button>
)}
```

---

## Smart mode switching

When users toggle between Encoder and Decoder, the tool swaps input and output for a continuous workflow:

```tsx
const handleModeChange = (newMode: "encode" | "decode") => {
    setMode(newMode);
    if (output && !error) {
        setInput(output);
        setOutput(""); // Will be filled by useEffect
    }
};
```

Encode something, switch to decode mode, and the encoded output becomes the new input — ready to be decoded back. The `!error` guard prevents swapping when the output is in an error state.

---

## What actually gets encoded?

`encodeURIComponent` follows RFC 3986. Here's what it does to different character types:

| Input | Output | Why |
|---|---|---|
| `hello world` | `hello%20world` | Spaces become `%20` |
| `price=10&qty=2` | `price%3D10%26qty%3D2` | `=` and `&` are reserved |
| `cafe` | `cafe` | ASCII letters pass through |
| `100%` | `100%25` | `%` itself must be escaped |
| `hello@world` | `hello%40world` | `@` is reserved |
| `emoji: 😀` | `emoji%3A%20%F0%9F%98%80` | UTF-8 bytes, percent-encoded |

The emoji example is worth noting. JavaScript strings are UTF-16, but `encodeURIComponent` converts to UTF-8 first, then percent-encodes each byte. The emoji (U+1F600) becomes 4 UTF-8 bytes: `F0 9F 98 80`, each prefixed with `%`.

---

## Error handling for invalid decode input

Decoding can fail. Here are inputs that will throw:

```js
decodeURIComponent("%")      // URIError: lone %
decodeURIComponent("%ZZ")    // URIError: invalid hex
decodeURIComponent("%C0%AF") // URIError: invalid UTF-8 sequence
```

The tool catches these and shows an inline error:

```tsx
<div className={cn(
    "rounded-xl border border-zinc-200",
    error && "border-red-500"
)}>
    <textarea readOnly value={output} />
    {error && (
        <div className="text-sm text-red-500">{error}</div>
    )}
</div>
```

---

## Copy to clipboard with visual feedback

```tsx
const [copied, setCopied] = useState(false);

const copyToClipboard = () => {
    navigator.clipboard.writeText(output);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
};

{copied ? <Check className="h-3 w-3" /> : <Copy className="h-3 w-3" />}
{copied ? "Copied" : "Copy"}
```

`navigator.clipboard.writeText` is async but we don't need to await it — the visual feedback is immediate, and clipboard writes virtually never fail in a secure context (HTTPS).

---

## The full component structure

```
UrlEncoderDecoder (client component)
├── State: input, output, mode, liveMode, copied, error
├── Controls bar
│   ├── Mode toggle (Encoder / Decoder)
│   ├── Live Mode checkbox
│   └── Clear All button
├── Split pane (responsive grid)
│   ├── Input textarea (editable)
│   └── Output textarea (read-only) + Copy button
└── Manual process button (only visible when live mode is off)
```

No external encoding libraries. No server calls. About 160 lines of code total.

---

Try it out: [URL Encoder/Decoder](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/)

---

*Originally published on [Hashnode](https://ultimatetools.hashnode.dev/building-a-url-encoder-decoder-with-live-mode-in-next-js). Part of [Ultimate Tools](https://ultimatetools.io/) — a free, privacy-first browser toolkit with 40+ tools.*