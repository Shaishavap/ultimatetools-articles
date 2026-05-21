# Building a URL Encoder/Decoder — encodeURIComponent vs encodeURI, When to Use Each, and Component-Level Encoding

The [URL Encoder/Decoder](https://ultimatetools.io/tools/text-tools/url-encoder/) at Ultimate Tools encodes and decodes URL strings. The non-obvious part isn't the encoding itself — it's choosing between `encodeURIComponent`, `encodeURI`, and a custom component-level encoder, and knowing when each one is correct.

---

## The Two Built-in Functions

JavaScript ships with two percent-encoding functions:

```javascript
encodeURI('https://example.com/path?q=hello world&lang=en')
// → 'https://example.com/path?q=hello%20world&lang=en'

encodeURIComponent('https://example.com/path?q=hello world&lang=en')
// → 'https%3A%2F%2Fexample.com%2Fpath%3Fq%3Dhello%20world%26lang%3Den'
```

The difference:

- **`encodeURI`** encodes characters that are invalid in a complete URL but leaves URL structure characters alone (`/ ? # : @ & = + $`). Use when encoding a whole URL.
- **`encodeURIComponent`** encodes everything except letters, digits, and `- _ . ! ~ * ' ( )`. Use when encoding a single URL component (a query parameter value, a path segment).

Using `encodeURIComponent` on a full URL encodes the `://` and all `/` characters — the URL becomes unusable. Using `encodeURI` on a query parameter value fails to encode `&` and `=` — the query string breaks.

---

## The URL Encoder Tool Logic

```typescript
type EncoderMode = 'full-url' | 'component' | 'query-string';

function encode(input: string, mode: EncoderMode): string {
  switch (mode) {
    case 'full-url':
      return encodeURI(input);

    case 'component':
      return encodeURIComponent(input);

    case 'query-string':
      return encodeQueryString(input);
  }
}

function decode(input: string): string {
  try {
    return decodeURIComponent(input.replace(/\+/g, ' '));
  } catch {
    // Invalid percent sequences — decode what we can
    return input.replace(/%([0-9A-Fa-f]{2})/g, (_, hex) => {
      try {
        return decodeURIComponent(`%${hex}`);
      } catch {
        return `%${hex}`; // leave malformed sequences as-is
      }
    });
  }
}
```

`decodeURIComponent` throws on malformed sequences like `%GG` (non-hex digits after `%`). The fallback decodes valid sequences character by character, leaving invalid ones untouched.

The `+` → space replacement is for `application/x-www-form-urlencoded` format (HTML form submission), where `+` represents a space in query strings. This is different from `%20`. For decoding generic URL-encoded strings, always replace `+` first.

---

## Query String Encoding

A query string like `name=Alice&city=New York&age=30` needs each key and value encoded independently:

```typescript
function encodeQueryString(raw: string): string {
  // Assume raw is "key=value&key=value" format
  return raw
    .split('&')
    .map(pair => {
      const eqIdx = pair.indexOf('=');
      if (eqIdx === -1) return encodeURIComponent(pair);

      const key   = pair.slice(0, eqIdx);
      const value = pair.slice(eqIdx + 1);
      return `${encodeURIComponent(key)}=${encodeURIComponent(value)}`;
    })
    .join('&');
}
```

`pair.indexOf('=')` finds the first `=` — values can contain `=` (e.g., base64 strings like `aGVsbG8=`). Using `split('=')` would break those.

The `&` between pairs is not encoded — it's the delimiter, not data. Each key and value is independently percent-encoded.

---

## URL Component Breakdown

For users who want to encode specific parts of a URL without touching others:

```typescript
interface UrlComponents {
  protocol: string;    // 'https:'
  host: string;        // 'example.com'
  pathname: string;    // '/path/to/page'
  search: string;      // '?q=hello world'
  hash: string;        // '#section'
}

function parseAndEncodeUrl(rawUrl: string): string {
  try {
    const url = new URL(rawUrl);

    // Path segments — encode each segment separately to preserve '/'
    const encodedPath = url.pathname
      .split('/')
      .map(segment => encodeURIComponent(segment))
      .join('/');

    // Query params — use URLSearchParams for correct encoding
    const params = new URLSearchParams(url.searchParams);
    const encodedSearch = params.toString() ? `?${params.toString()}` : '';

    return `${url.protocol}//${url.host}${encodedPath}${encodedSearch}${url.hash}`;
  } catch {
    // Not a valid URL — fall back to encodeURI
    return encodeURI(rawUrl);
  }
}
```

`new URL(rawUrl)` parses the URL into components. `URLSearchParams.toString()` handles query parameter encoding correctly — it uses `+` for spaces (application/x-www-form-urlencoded) rather than `%20`. Check what your target system expects: APIs usually want `%20`, HTML forms want `+`.

Encoding path segments individually (splitting on `/`) preserves the path separator. `encodeURIComponent('/path/to/page')` would encode the slashes, producing `%2Fpath%2Fto%2Fpage` — a single path segment string, not a path.

---

## Characters by Category

```typescript
// Characters that are NEVER encoded by either function
const ALWAYS_SAFE = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_.!~*\'()';

// Characters encoded by encodeURIComponent but not encodeURI
// (URL structure characters — valid in a URL but not in a component value)
const URL_STRUCTURE = ':/?#[]@!$&\'()*+,;=';

// Characters that both encode (truly unsafe in URLs)
const ALWAYS_UNSAFE = ' "<>{}|\\^`';
```

For reference when debugging unexpected encoding behavior: if a character is in `URL_STRUCTURE`, it will be encoded by `encodeURIComponent` but not `encodeURI`.

---

## Implementation in React

```tsx
function UrlEncoderTool() {
  const [input, setInput] = useState('');
  const [mode, setMode] = useState<EncoderMode>('component');
  const [output, setOutput] = useState('');

  const handleEncode = () => setOutput(encode(input, mode));
  const handleDecode = () => setOutput(decode(input));

  return (
    <div>
      <textarea value={input} onChange={e => setInput(e.target.value)} placeholder="Enter URL or component..." />
      <select value={mode} onChange={e => setMode(e.target.value as EncoderMode)}>
        <option value="component">Component (encodeURIComponent)</option>
        <option value="full-url">Full URL (encodeURI)</option>
        <option value="query-string">Query String</option>
      </select>
      <button onClick={handleEncode}>Encode</button>
      <button onClick={handleDecode}>Decode</button>
      <textarea value={output} readOnly />
    </div>
  );
}
```

Live encoding (on every keystroke) works for short strings. For large inputs, debounce the encode call by 150–200ms to avoid blocking the input field on each character.

---

The full [URL Encoder/Decoder](https://ultimatetools.io/tools/text-tools/url-encoder/) — component mode, full URL mode, query string mode, decode with malformed-sequence tolerance — is live at Ultimate Tools. Paste your URL and encode or decode it instantly in the browser.