# What %20, %3F, and %26 Mean in a URL — URL Encoding Explained Simply

You copy a URL from your browser and paste it somewhere. It looks like this:

```
https://example.com/search?q=hello%20world%3F&lang=en%26fr
```

What are `%20`, `%3F`, and `%26`? Why does the URL have percent signs and numbers in the middle of it?

This is URL encoding — and once you understand it, those weird percent sequences become instantly readable.

---

## Why URLs can't have certain characters

A URL has a strict structure:

```
https://example.com/path?key=value&key2=value2
```

Characters like `?`, `&`, `=`, `/`, and `#` have specific jobs in that structure. `?` marks the start of query parameters. `&` separates parameters. `=` connects a key to its value.

If your actual data contains one of those characters — say, you're searching for the phrase `cats & dogs` — the URL can't just contain a literal `&` inside the value. That `&` would be interpreted as a parameter separator, not as part of your search phrase.

The solution: encode any character that has special meaning (or isn't safe in a URL) as a `%` followed by its two-digit hex code. Space becomes `%20`. Ampersand becomes `%26`. Question mark becomes `%3F`.

---

## The most common encoded characters

| Character | Encoded | Where you see it |
|---|---|---|
| Space | `%20` | Search queries, filenames with spaces |
| `?` | `%3F` | When a `?` is part of the value, not the separator |
| `&` | `%26` | "cats & dogs" in a query value |
| `=` | `%3D` | When `=` is part of the value |
| `#` | `%23` | Hashtags, anchors inside values |
| `/` | `%2F` | Slashes inside a path value |
| `+` | `%2B` | When a `+` is literal (not a space shorthand) |
| `@` | `%40` | Email addresses in URLs |
| `:` | `%3A` | Colons inside values |

The pattern is always the same: `%` + the two-character hexadecimal code for the character's ASCII value. Space is ASCII 32, which is `20` in hex. Hence `%20`.

---

## %20 vs + for spaces

You'll sometimes see spaces encoded as `+` instead of `%20`:

```
https://example.com/search?q=hello+world
https://example.com/search?q=hello%20world
```

Both mean the same thing in a query string — a space. `+` is shorthand for space only inside query parameters (the part after `?`). In the path part of a URL (before `?`), `+` is a literal plus sign. `%20` is unambiguous in both parts.

When in doubt: `%20` is always correct for a space. `+` only works in query strings.

---

## When the browser does it automatically

You never type `%20` yourself in the address bar. The browser encodes automatically when you:

- Type a search into Google and look at the resulting URL
- Paste text with spaces or special characters into a URL bar
- Submit a form with text inputs

The browser takes what you typed, encodes it, and sends the encoded URL to the server. The server decodes it and reads the original text.

When you share a URL or link to it from code, the encoding should already be there.

---

## When you need to encode manually

In JavaScript, two functions handle this:

```javascript
// encodeURIComponent — encodes everything except letters, digits, -, _, ., ~
encodeURIComponent("cats & dogs?") // → "cats%20%26%20dogs%3F"

// decodeURIComponent — reverses the encoding
decodeURIComponent("cats%20%26%20dogs%3F") // → "cats & dogs?"
```

Use `encodeURIComponent` when you're building URLs in code and need to safely insert a user-provided value into a query parameter.

Don't use `encodeURI` (without "Component") for query values — it intentionally leaves `?`, `&`, `=`, and other URL-structural characters unencoded, so it won't protect your values correctly.

---

## How to decode a URL you received

If someone sends you a URL full of `%xx` sequences and you want to read it clearly:

1. Go to [URL Encoder/Decoder at Ultimate Tools](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/)
2. Paste the encoded URL into the input
3. Click **Decode**
4. Read the decoded, human-readable version

It handles the full URL or just a fragment — whatever you paste.

---

## Quick decode reference

If you just need to read a specific `%xx` sequence, here's the shortcut: the two hex digits after `%` are the character's ASCII code in hexadecimal.

`%20` → hex 20 → decimal 32 → ASCII space  
`%3F` → hex 3F → decimal 63 → ASCII `?`  
`%26` → hex 26 → decimal 38 → ASCII `&`

Or just paste the whole URL into the decoder above — it's faster than doing hex math.

---

**[URL Encoder/Decoder at Ultimate Tools](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/)** — encode or decode URLs instantly. Free, runs in your browser, no signup.