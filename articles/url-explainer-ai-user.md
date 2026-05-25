# What's Actually in That Link? A Free AI URL Explainer for UTM Tags, Tracking IDs, and Suspicious Domains

Every URL you click carries extra information beyond the page address. UTM parameters tell analytics tools where the visitor came from. Affiliate IDs earn someone a commission. Click IDs feed advertising platforms. OAuth callbacks tell the auth server where to send you next. Most people never see what those parameters do — they just click and the URL bar fills with strings of characters that look like noise.

There's now a [free AI URL explainer at Ultimate Tools](https://ultimatetools.io/tools/security-tools/url-explainer/) that decodes any URL you paste. It breaks the URL into structural parts (protocol, subdomain, domain, path), explains every query parameter in plain English, and flags any signs that the URL might be a phishing or scam attempt — punycode lookalikes, missing HTTPS, embedded credentials, hidden shortener targets.

---

## What the Tool Does in One Pass

You paste a URL. You click Explain. You get four panels back:

1. **Plain-English summary** — one sentence describing what the link does and what a click will trigger.
2. **Safety flags** — a list of structural traits that suggest something is off (only shown if there's anything to flag).
3. **URL structure** — protocol, subdomain, domain, port, path, fragment — each separated and labeled.
4. **Query parameters** — every key/value pair, each with a one-line explanation of its purpose.

For a marketing URL like `https://shop.example.com/sale?utm_source=newsletter&utm_medium=email&utm_campaign=spring2026&aff=jane123`, the parameter panel shows:

- `utm_source = newsletter` — Identifies the traffic source. Set by the link owner for analytics.
- `utm_medium = email` — Identifies the marketing medium.
- `utm_campaign = spring2026` — Names the campaign this link belongs to.
- `aff = jane123` — Affiliate ID. The person sending you the link earns a commission if you buy.

That last one is the kind of thing most people miss.

---

## How the Tool Decides What Each Parameter Means

The tool ships with a list of 40+ commonly-used parameters and their purposes — the UTM family, ad-network click IDs (`gclid`, `fbclid`, `msclkid`, `yclid`), email tracking (`mc_cid`, `mc_eid`), affiliate tags (`aff`, `affid`, `tag`), OAuth flow parameters (`code`, `state`, `token`, `next`, `redirect`), and platform-specific share IDs (`igshid` for Instagram, `si` for Spotify/YouTube, `spm` for AliExpress/Taobao).

For any parameter NOT in that list, the tool sends just the unknown parameters (plus the URL itself) to Google Gemma (the `gemma-3-27b-it` model) and asks for a one-sentence purpose for each. The output is constrained: max 18 words, no preamble, just the purpose.

This split matters. The known list is fast, exact, and uses no API quota. Gemma is reserved for the unknown stuff and the plain-English summary. Together they cover the realistic universe of URLs without burning the daily AI budget on parameters that have stable, well-documented meanings.

---

## Safety Flags

The structural safety checks happen entirely in the browser:

- **Non-HTTPS** → flagged ("data is not encrypted in transit").
- **Embedded credentials** (`https://user:password@example.com/`) → flagged ("almost always malicious or misconfigured").
- **IP-literal hosts** (`http://203.0.113.42/login`) → flagged ("uncommon outside internal systems").
- **Punycode-encoded domains** (`xn--apple-something.com`) → flagged. Punycode is what browsers use to display internationalized domain names; it's also how phishers register `аpple.com` (Cyrillic 'а') as a lookalike of `apple.com` (Latin 'a').

These flags appear in their own orange panel labeled "Things to know" — they don't block usage, they just surface red flags so you can make an informed decision before clicking.

---

## What the Tool Doesn't Do (Deliberately)

It does **not** follow the URL to find the final destination after redirects. Following a suspicious URL server-side would expose the server's IP and any cookies the URL sets — exactly the bait that phishing landing pages are testing for. The tool analyses the URL string you paste, not the page it leads to.

It does **not** rate the URL against a threat-intelligence feed (VirusTotal, Google Safe Browsing, etc). The safety flags are pattern-based, not reputation-based. A known phishing URL with a clean structure won't be flagged on reputation; a brand-new URL with a suspicious structure WILL be flagged on structure. Use the explainer as a first-pass filter, not a vulnerability scanner.

It does **not** fetch the page title or favicon. Same reason — no outbound request to the URL itself.

---

## Use Cases

- **Suspicious email link triage** — an email looks like it's from a known service but the link feels off. Paste it, check the domain breakdown, look at the safety flags. Takes 10 seconds.
- **Marketing campaign audit** — verify your team's UTM tagging across a list of recent links. Paste each one, confirm `utm_source`, `utm_medium`, and `utm_campaign` are all populated correctly.
- **Affiliate-link disclosure** — see whether a "recommendation" link you received is actually an affiliate link. Useful for transparency, blogger ethics, or just understanding what kind of incentive structure is behind the recommendation.
- **OAuth / callback inspection** — pasting a `?code=...&state=...&next=/dashboard` URL shows you exactly what redirect target the auth server is being told to use after login. Useful for debugging your own auth flows.
- **Receipt and order link audit** — order confirmation emails sometimes embed your email address or order ID in the URL. The tool surfaces these so you know what you're sharing if you forward the link to someone else.
- **Learning how the web tracks you** — paste links from your own browsing history and see the marketing layer made visible. A great exercise for understanding why your inbox suddenly has ads for a product you only mentioned in a Slack DM.

---

## Privacy

- URL parsing, structural breakdown, safety flag detection, and the known-parameter list — all run in your browser. Nothing leaves your device for those.
- The URL string is sent to Google Gemma **only** when the tool needs the AI portion: identifying unfamiliar parameters or generating the plain-English summary.
- Don't paste URLs containing private session tokens, one-time login codes, or anything you wouldn't be comfortable sharing.

The disclosure is right above the input field, not buried in a footer.

---

## Cost and Rate Limits

The tool is free. Per-IP rate limit is 15 URL explanations per minute. Daily quota is shared across all visitors and resets daily. Under normal load you'll never see the limit.

---

## Related Tools

- [QR code scanner with AI safety check](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/) — same kind of analysis applied to URLs decoded from QR codes
- [free password generator with passphrase mode and strength checker](https://ultimatetools.io/tools/security-tools/password-generator/) — for the password rotation that follows a click you shouldn't have made
- [URL encoder and decoder for safe web URLs](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/) — for converting between percent-encoded and human-readable URL forms

---

The reason URL tracking works as well as it does is that URLs look like noise to most people. Once you see what's actually in the query string — the campaign tag, the affiliate ID, the click ID, the redirect target — the noise resolves into a story about who's measuring what and who's getting paid. The tool doesn't change the URL. It just makes it readable.

**[Decode any URL with the free AI URL explainer →](https://ultimatetools.io/tools/security-tools/url-explainer/)**