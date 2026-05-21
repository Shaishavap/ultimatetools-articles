# Building a Base64 Encoder/Decoder with File Support in Next.js

Base64 is everywhere — data URLs, email attachments, API payloads, JWTs. But the browser's built-in `btoa()` and `atob()` have a well-known limitation: they choke on Unicode. I built a Base64 tool that handles UTF-8 text, file uploads, and binary downloads — all client-side. Here's how it works.

The live tool is at [Base64 Encoder/Decoder](https://ultimatetools.io/tools/coding-tools/base64-encoder-decoder/).

---

## The UTF-8 problem with btoa

`btoa()` only accepts characters in the Latin-1 range (U+0000 to U+00FF). Try encoding anything outside that range and it throws:

```js
btoa("hello")     // "aGVsbG8=" ✅
btoa("hello 🌍")  // DOMException: The string contains characters outside Latin-1 ❌
```

The standard workaround is to pipe through `encodeURIComponent` and `unescape` first:

```js
// Encode: string → UTF-8 bytes → Base64
btoa(unescape(encodeURIComponent("hello 🌍")))
// "aGVsbG8g8J+MjQ=="  ✅

// Decode: Base64 → UTF-8 bytes → string
decodeURIComponent(escape(atob("aGVsbG8g8J+MjQ==")))
// "hello 🌍"  ✅
```

Here's what happens step by step:
1. `encodeURIComponent("hello 🌍")` → `"hello%20%F0%9F%8C%8D"` (UTF-8 percent-encoded)
2. `unescape(...)` → converts each `%XX` to a Latin-1 character (byte-level mapping)
3. `btoa(...)` → encodes those Latin-1 characters to Base64

Yes, `unescape` and `escape` are deprecated. They still work in every browser and they're the simplest way to bridge the gap between UTF-8 and Latin-1. The alternative is manual `TextEncoder`/`TextDecoder` with `Uint8Array` — more explicit, but more code for the same result (shown at the end).

---

## The core processing function

```tsx
const process = (value: string, currentMode: Mode) => {
    setInput(value);
    setError("");
    if (!value.trim()) { setOutput(""); return; }

    try {
        if (currentMode === "encode") {
            setOutput(btoa(unescape(encodeURIComponent(value))));
        } else {
            setOutput(decodeURIComponent(escape(atob(value.trim()))));
        }
    } catch {
        setOutput("");
        setError(currentMode === "encode"
            ? "Encoding failed."
            : "Invalid Base64 string.");
    }
};
```

This runs on every keystroke. Since `btoa`/`atob` are native browser functions, encoding is essentially instant even for large strings.

The `try/catch` is critical. `atob` throws on invalid Base64 input — characters outside `A-Za-z0-9+/=` or incorrect padding. The catch block shows a clear error message instead of crashing.

---

## File-to-Base64 with FileReader

The tool supports uploading any file and converting it to Base64. Useful for embedding images in CSS, sending binary data through JSON APIs, or debugging data URLs.

```tsx
const handleFileUpload = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;
    setFileName(file.name);

    const reader = new FileReader();
    reader.onload = () => {
        const dataUrl = reader.result as string;
        // dataUrl = "data:image/png;base64,iVBORw0KGgo..."
        const base64 = dataUrl.split(",")[1];
        setInput(base64);
        setOutput(base64);
        setMode("decode");
    };
    reader.readAsDataURL(file);
    e.target.value = ""; // Allow re-uploading same file
};
```

Key decisions:
- **`readAsDataURL`** instead of `readAsArrayBuffer` — gives us the Base64 string directly, no manual conversion needed
- **`split(",")[1]`** — strips the `data:mime/type;base64,` prefix to get raw Base64
- **Switches to decode mode** — after upload, the Base64 is in the input field, ready to be decoded or copied
- **`e.target.value = ""`** — resets the file input so uploading the same file again triggers the `onChange` event

---

## Downloading decoded binary

When you paste Base64 of a file (image, PDF, zip) and decode it, you want the actual file back:

```tsx
const handleDownload = () => {
    if (!output) return;
    try {
        const binary = atob(output.trim());
        const bytes = new Uint8Array(binary.length);
        for (let i = 0; i < binary.length; i++) {
            bytes[i] = binary.charCodeAt(i);
        }
        const blob = new Blob([bytes]);
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = "decoded-file";
        a.click();
        URL.revokeObjectURL(url);
    } catch {
        setError("Cannot download — output is not valid Base64 binary.");
    }
};
```

The flow:
1. `atob(output)` → decodes Base64 to a binary string (each char is one byte)
2. `charCodeAt(i)` → converts each character to its byte value (0–255)
3. `Uint8Array` → creates a proper byte array
4. `Blob` + `createObjectURL` → creates a downloadable file in memory
5. `revokeObjectURL` — frees the memory immediately after

---

## Mode switching without data loss

```tsx
const handleModeSwitch = (newMode: Mode) => {
    setMode(newMode);
    setFileName("");
    process(input, newMode);
};
```

Switching between encode and decode re-processes the current input with the new mode. If you have text in the input and switch from encode to decode, it immediately tries to decode that text as Base64. Simple and predictable.

---

## Edge cases worth knowing

**Whitespace in Base64 input:** The tool trims before decoding. Base64 copied from emails or formatted JSON often has newlines — `atob` doesn't handle those, so `.trim()` prevents unnecessary errors.

**Large files:** `FileReader.readAsDataURL` loads the entire file into memory as a Base64 string. For a 10MB file, that's ~13.3MB of Base64 text in a textarea. It works but the browser will slow down.

**The modern UTF-8 approach** (if you want to avoid deprecated functions):

```js
// Modern encode
const encode = (str) => {
    const bytes = new TextEncoder().encode(str);
    const binary = String.fromCharCode(...bytes);
    return btoa(binary);
};

// Modern decode
const decode = (b64) => {
    const binary = atob(b64);
    const bytes = Uint8Array.from(binary, c => c.charCodeAt(0));
    return new TextDecoder().decode(bytes);
};
```

Same result, more explicit about the UTF-8 conversion step.

---

Try it out: [Base64 Encoder/Decoder](https://ultimatetools.io/tools/coding-tools/base64-encoder-decoder/)

No npm packages. No server. Just the browser APIs doing what they were built for.

---

*Originally published on [Hashnode](https://ultimatetools.hashnode.dev/building-a-base64-encoder-decoder-with-file-support-in-next-js). Part of [Ultimate Tools](https://ultimatetools.io/) — a free, privacy-first browser toolkit with 40+ tools.*