# How to URL Encode and Decode Text Online for Free — Fix %20, %3F, %26, and Special Characters

You paste a URL into a message and the recipient gets a broken link. You copy a query parameter and it contains `%20` everywhere a space should be. You build a web form and special characters in the input break your API call.

URL encoding and decoding fixes all of these. Here's how to do it instantly at [Ultimate Tools](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/).

---

## Why URLs can't contain certain characters

URLs are restricted to a specific set of safe characters: letters (A–Z, a–z), digits (0–9), and a handful of symbols (`-`, `_`, `.`, `~`). Everything else — spaces, ampersands, question marks, equals signs, slashes, Unicode characters — must be encoded before appearing in a URL.

The encoding format is **percent-encoding**: each unsafe character is replaced with a `%` followed by its two-digit hexadecimal ASCII code.

| Character | Encoded | Reason reserved |
|---|---|---|
| Space | `%20` | Terminates URLs in many parsers |
| `&` | `%26` | Separates query parameters |
| `=` | `%3D` | Separates key from value in query params |
| `?` | `%3F` | Starts the query string |
| `#` | `%23` | Starts a fragment identifier |
| `+` | `%2B` | Used as space in some encoding formats |
| `/` | `%2F` | Path separator |

---

## When you need URL encoding

**Passing text as a query parameter**

`https://example.com/search?q=hello world` — the space breaks the URL. Encoded: `?q=hello%20world`.

**Embedding a URL inside another URL**

`https://redirect.com/?url=https://example.com/path?key=value` — the inner `?` and `=` break the outer URL's query string. Encode the inner URL: `?url=https%3A%2F%2Fexample.com%2Fpath%3Fkey%3Dvalue`.

**Submitting form data with special characters**

Names with apostrophes, addresses with `&`, email subjects with `+` or `?` — all need encoding before being sent in a URL.

**Working with non-ASCII characters**

Japanese, Arabic, Chinese, and emoji characters must be UTF-8 encoded then percent-encoded. `こんにちは` becomes `%E3%81%93%E3%82%93%E3%81%AB%E3%81%A1%E3%81%AF`.

---

## When you need URL decoding

**Reading a URL someone sent you**

`https://example.com/search?q=pdf%20to%20word%20converter` → decoded query: "pdf to word converter". Instantly readable.

**Debugging an API call**

API request URLs in logs are often fully encoded. Decoding makes the parameters human-readable without manually translating each `%XX`.

**Copying text from a URL**

Browser address bars sometimes encode text you've typed. Decoding gives you back the original string.

---

## How to use the encoder/decoder

1. Go to [URL Encoder/Decoder at Ultimate Tools](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/)
2. Paste your text or URL into the input box
3. Click **Encode** or **Decode**
4. The result appears instantly
5. Click **Copy** to copy to clipboard

The tool handles both directions — encode plain text to a URL-safe format, or decode an encoded URL back to readable text.

---

## Encoding vs decoding — which one do you need?

**You have plain text and need a URL-safe version** → Encode

Example: You're building a share link and need to pass "Hello World & More" as a parameter → encode it → `Hello%20World%20%26%20More`

**You have a URL with %XX sequences and want to read it** → Decode

Example: You see `q=what+is+%22url+encoding%22%3F` in a log → decode it → `q=what is "url encoding"?`

---

## The difference between `%20` and `+`

Both represent a space, but in different contexts:

- `%20` — the universal percent-encoded space, valid everywhere in a URL
- `+` — shorthand for space only in `application/x-www-form-urlencoded` data (HTML form submissions)

If you're encoding a URL query parameter manually, use `%20`. The `+` form can cause issues outside of form data contexts.

---

**[URL Encoder/Decoder at Ultimate Tools](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/)** — encode or decode any text instantly, free, no signup.