# URL Query String Parser Online Free — Extract UTM Params and Decode Any URL

You get a link like this in your analytics dashboard:

```
https://example.com/blog?utm_source=newsletter&utm_medium=email&utm_campaign=may-launch&ref=homepage&page=2
```

Figuring out what each parameter does requires either reading the URL character by character or copying each key-value pair manually. Neither is fun when you're debugging a campaign or checking an API response.

The [URL query string parser online free](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/) at Ultimate Tools does it in one paste. Switch to the **Parse URL** tab, paste your URL, and every component breaks out into a clean table instantly.

---

## What the Parser Shows You

Paste any full URL and the tool splits it into:

| Component | Example |
|---|---|
| Protocol | `https` |
| Domain | `example.com` |
| Path | `/blog` |
| Fragment | `#section` |
| Query parameters | Each key + value as separate rows |

Every query parameter gets its own row. For the URL above, you'd see:

| Key | Value |
|---|---|
| utm_source | newsletter |
| utm_medium | email |
| utm_campaign | may-launch |
| ref | homepage |
| page | 2 |

Click the copy icon next to any value to copy it to clipboard without touching the surrounding text.

---

## When Is This Actually Useful?

**Debugging UTM parameters.** Marketing links get long and messy. Paste a campaign URL into the parser to instantly confirm which utm_source, utm_medium, and utm_campaign values are actually being tracked — before the campaign goes live.

**Checking API responses.** REST APIs often return URLs with embedded parameters in the response body. Parsing them manually is error-prone. Paste the URL and see every parameter clearly.

**Decoding redirect chains.** Affiliate links and tracking URLs nest parameters inside other parameters (often percent-encoded). Paste the outer URL, copy a specific encoded value, then paste that into the Decode tab to unwrap it.

**Extracting OAuth callback parameters.** OAuth flows return `code`, `state`, and `error` parameters in a redirect URL. If something fails, paste the full callback URL to see exactly what was returned.

**Verifying canonical and pagination links.** Copy a canonical URL from your CMS and parse it to confirm the `page` or `lang` parameter values are what your SEO team expects.

---

## Encode and Decode Are Still There

The Parse URL tab is an addition — it doesn't replace the Encode and Decode tabs.

**Encoder tab:** Paste raw text with special characters (`&`, `=`, `?`, spaces) → get the percent-encoded version safe to embed in a URL.

**Decoder tab:** Paste a percent-encoded string → get the human-readable version back (`%20` → space, `%26` → `&`, etc.).

**Parse URL tab:** Paste a full URL → get every component in a table.

All three work client-side in your browser. Nothing is sent to any server.

---

## How to Use the URL Parser

1. Open the [URL Encoder / Decoder & Query String Parser](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/)
2. Click the **Parse URL** tab (third tab, after Encoder and Decoder)
3. Paste any full URL into the input box
4. The table populates instantly — protocol, domain, path, fragment, and all query parameters as individual key-value rows
5. Click the copy icon next to any value to copy it

If the URL is malformed or not parseable, an error message appears beneath the input.

---

## Related Tools

- [minify javascript css html online free no ads](https://ultimatetools.io/tools/coding-tools/code-beautifier/) — compress or beautify the JavaScript and CSS files whose URLs you're debugging — same browser-only, no-upload approach
- [format and validate JSON online free](https://ultimatetools.io/tools/text-tools/json-formatter/) — when your URL's query parameter value is a JSON-encoded string, paste the decoded value here to format and validate it
- [generate bulk qr codes from csv free zip download](https://ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/) — batch-generate QR codes for the URLs you've been parsing and debugging — CSV input, ZIP output

---

**[Parse any URL and extract query parameters free →](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/)**