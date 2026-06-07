# Unix Timestamp to Date — Convert Epoch Time Free (Seconds & ms)

You're staring at `1749312000` in a log line, a database row, or a JWT's `exp` claim, and you need to know what actual date that is. Converting a **Unix timestamp to a date** is one of those things you do constantly and never want to do by hand. Here's what the number means, the one gotcha that trips everyone up, and the fastest way to convert it either direction.

## What a Unix timestamp actually is

A Unix timestamp (or *epoch time*) is the number of seconds since **00:00:00 UTC on 1 January 1970**, not counting leap seconds. That's it. It's popular because it's compact, sortable, and timezone-independent — a single integer that means the exact same instant everywhere on Earth. The timezone only enters the picture when you convert it back to a human-readable date.

## The gotcha: seconds vs milliseconds

This is the mistake that wastes ten minutes every time. Classic Unix time is in **seconds** — a 10-digit number like `1749312000`. But JavaScript's `Date.now()` and many APIs return **milliseconds** — a 13-digit number like `1749312000000`.

Feed milliseconds into something expecting seconds (or vice versa) and you land thousands of years in the future or stuck in 1970. The tell is the digit count: 10 digits ≈ seconds, 13 digits ≈ milliseconds (for dates around now).

## The fastest way: convert in the browser

For a quick lookup you don't want to write code or fight timezones. The [free Unix timestamp converter](https://ultimatetools.io/tools/coding-tools/timestamp-converter/) handles both directions:

- **Paste a timestamp** — it auto-detects seconds vs milliseconds and shows the local time, UTC, ISO 8601, day of week, and relative time ("3 hours ago")
- **Pick a date** — get the epoch value back in both seconds and milliseconds, ready to copy
- **A live current timestamp** ticks at the top, so grabbing "unix time now" is one click
- **Runs entirely in your browser** — nothing is sent to a server

Paste the number, read the date, copy what you need. Done.

## Doing it in code

If you'd rather do it inline, the conversions are short — just mind the units:

```js
// Seconds → Date (multiply by 1000, because JS uses ms)
new Date(1749312000 * 1000).toISOString(); // "2025-06-07T..."

// Milliseconds → Date (use directly)
new Date(1749312000000).toUTCString();

// Date → Unix seconds
Math.floor(Date.now() / 1000);
```

The `* 1000` on the first line is the single most common bug in timestamp handling. If your date looks absurd, that factor of 1000 is almost always why.

## Why local time and UTC differ

A timestamp has no timezone — it's always a UTC instant. When you convert it, "local" applies your machine's timezone offset while "UTC" and the ISO 8601 string show universal time. That's why the same timestamp shows a different wall-clock time on your laptop than on a server in another region. For storing and exchanging dates between systems, prefer **ISO 8601** (e.g. `2026-06-07T12:00:00.000Z`) — the trailing `Z` makes the UTC intent explicit and removes the ambiguity.

## A common real use: checking a JWT's expiry

JWTs store `iat` (issued at) and `exp` (expires) as Unix timestamps in seconds. When a token "mysteriously" stops working, decode it and convert `exp` to a date — nine times out of ten it simply expired. Decode the payload, drop the `exp` number into the converter, and you have your answer in seconds.

## One honest caveat

Relative time ("3 hours ago") and the local-time line depend on your device's clock and timezone being correct — if your system clock is off, those readouts will be too. The raw timestamp, UTC, and ISO 8601 values are absolute and unaffected, so when precision matters, trust those over the relative description.

## Related Tools

- [format and validate a JSON API response online free](https://ultimatetools.io/tools/text-tools/json-formatter/) — pretty-print the response so you can find the timestamp field fast
- [decode a JWT or Base64 token to read its claims](https://ultimatetools.io/tools/coding-tools/base64-encoder-decoder/) — pull the `exp` and `iat` epoch values out of a token to convert them
- [decode URL-encoded query parameters online](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/) — unescape the encoded values that often sit next to timestamps in API requests

Stop second-guessing whether it's seconds or milliseconds. **[Open the free Unix timestamp converter →](https://ultimatetools.io/tools/coding-tools/timestamp-converter/)** — paste an epoch value, get local, UTC, ISO 8601 and relative time instantly, or convert a date back. Nothing uploaded.