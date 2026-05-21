# Building a Client-Side JSON Formatter and Validator in Next.js

JSON tooling is one of those things every developer reaches for constantly — debugging API responses, reading config files, preparing request bodies. I built a JSON formatter and validator as part of [Ultimate Tools](https://ultimatetools.io/tools/text-tools/json-formatter/) and wanted to share how it works under the hood.

Full feature set: real-time formatting, inline validation with error messages, 2-space / 4-space indent toggle, minify mode, one-click copy. All client-side. No API calls. No dependencies beyond React.

---

## The Core: JSON.parse + JSON.stringify

The entire formatting logic is two native browser functions:

```ts
const format = (input: string, indent: number): string => {
  const parsed = JSON.parse(input);
  return JSON.stringify(parsed, null, indent);
};
```

`JSON.parse` validates and parses. `JSON.stringify` with the third argument (indent) produces formatted output. That's it. The indent argument accepts a number (spaces) or a string (e.g. `'\t'` for tabs).

---

## Real-Time Validation

The formatter validates as the user types. We wrap `JSON.parse` in a try/catch and extract the error message:

```ts
const validateJson = (input: string): { valid: boolean; error: string | null } => {
  if (!input.trim()) return { valid: false, error: null };

  try {
    JSON.parse(input);
    return { valid: true, error: null };
  } catch (e) {
    return {
      valid: false,
      error: e instanceof SyntaxError ? e.message : 'Invalid JSON',
    };
  }
};
```

The `SyntaxError.message` from `JSON.parse` is surprisingly informative:

```
Unexpected token } in JSON at position 42
Expected ',' or ']' after array element in JSON at position 18
```

We surface this directly in the UI. No need to write our own error messages — the native parser does it for us.

---

## The State Model

```ts
type FormatterState = {
  input: string;
  output: string;
  error: string | null;
  isValid: boolean;
  indent: 2 | 4;
  mode: 'format' | 'minify';
};
```

On every input change, we run validation and formatting:

```ts
const handleInput = (value: string) => {
  setInput(value);

  if (!value.trim()) {
    setOutput('');
    setError(null);
    setIsValid(false);
    return;
  }

  const { valid, error } = validateJson(value);

  if (!valid) {
    setError(error);
    setIsValid(false);
    setOutput('');
    return;
  }

  setIsValid(true);
  setError(null);

  const parsed = JSON.parse(value);
  const formatted =
    mode === 'minify'
      ? JSON.stringify(parsed)
      : JSON.stringify(parsed, null, indent);

  setOutput(formatted);
};
```

Key decision: when JSON is invalid, we clear the output rather than showing a partial result. This avoids confusing the user with stale formatted output while they are mid-edit.

---

## Indent Toggle

When the user changes the indent, we reformat the existing valid JSON immediately:

```ts
const handleIndentChange = (newIndent: 2 | 4) => {
  setIndent(newIndent);
  if (isValid && input) {
    const parsed = JSON.parse(input);
    setOutput(JSON.stringify(parsed, null, newIndent));
  }
};
```

No re-validation needed — if the JSON was valid before, it is still valid.

---

## Minify Mode

```ts
const minified = JSON.stringify(JSON.parse(input));
```

`JSON.stringify` without the third argument produces compact output — no spaces, no newlines. We toggle between format and minify by changing the `mode` state and re-running the output calculation.

---

## Performance: Debouncing

For large JSON inputs (a 500KB API response), running `JSON.parse` on every keystroke is wasteful. We debounce:

```ts
const process = useDebouncedCallback((value: string) => {
  // validation + formatting logic
}, 150);
```

150ms debounce means the formatter waits for the user to pause typing before processing. For typical JSON sizes this is imperceptible.

---

## What JSON.parse Does NOT Catch

**Duplicate keys** — `{"a": 1, "a": 2}` does NOT throw. `JSON.parse` silently keeps the last value. This is technically invalid JSON per the spec but most parsers accept it. We don't flag it.

**Number precision** — Large integers like `9007199254740993` lose precision silently because JavaScript's `Number` type cannot represent them exactly. A known limitation of `JSON.parse`.

---

## Full Component

```ts
'use client';

export default function JsonFormatter() {
  const [input, setInput] = useState('');
  const [output, setOutput] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [isValid, setIsValid] = useState(false);
  const [indent, setIndent] = useState<2 | 4>(2);
  const [mode, setMode] = useState<'format' | 'minify'>('format');
  const [copied, setCopied] = useState(false);

  const process = useDebouncedCallback((value: string) => {
    if (!value.trim()) {
      setOutput(''); setError(null); setIsValid(false); return;
    }
    try {
      const parsed = JSON.parse(value);
      setIsValid(true);
      setError(null);
      setOutput(
        mode === 'minify'
          ? JSON.stringify(parsed)
          : JSON.stringify(parsed, null, indent)
      );
    } catch (e) {
      setIsValid(false);
      setError(e instanceof SyntaxError ? e.message : 'Invalid JSON');
      setOutput('');
    }
  }, 150);

  return ( /* UI */ );
}
```

No API routes, no server actions, no external dependencies for the core logic. Just `JSON.parse` and `JSON.stringify`. The whole thing is about 150 lines.

---

Try it out: [JSON Formatter](https://ultimatetools.io/tools/text-tools/json-formatter/)

---

*Originally published on [Hashnode](https://ultimatetools.hashnode.dev/building-a-client-side-json-formatter-and-validator-in-next-js). Part of [Ultimate Tools](https://ultimatetools.io/) — a free, privacy-first browser toolkit with 40+ tools.*