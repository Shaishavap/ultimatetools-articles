# Adding AI to a Privacy-First Tool Site Without Breaking the Privacy Promise — the Architecture Decisions

I run a utility-tools site that has always claimed "everything runs in your browser, nothing leaves your device". For two years that claim was true across every feature — the PDF tools, image processing, word counter, QR generator, all of it. Then I wanted to add AI.

AI features genuinely cannot run client-side. The model weights are tens of gigabytes; you cannot ship them to a browser. The moment AI is involved, some user data has to leave the device. The question is what data, when, and how clearly you disclose it to the user.

This post is the architecture decision log for adding three AI features to the site — an [AI line-item writer for invoices/quotes/receipts](https://ultimatetools.io/tools/misc-tools/invoice-generator/), an [AI safety check for QR code scanning](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/), and a [URL Explainer that decodes UTM tags and tracking parameters](https://ultimatetools.io/tools/security-tools/url-explainer/) — without invalidating the privacy promise on the 50+ existing local-only features.

---

## The Trust Boundary Pattern

The architecture I landed on is "trust boundary explicit per feature". Every tool page falls into one of three categories:

1. **Fully local** — input, processing, and output all stay in the browser. The PDF merger, image converter, JSON formatter, QR generator. The user's data never crosses any network boundary. This is the historical default.

2. **Local-only with explicit AI escape hatch** — the core feature is local, but a single optional button (clearly labeled, disclosed inline) sends user-typed data to an AI model. Example: the Word Counter. Character counting, syllable counting, Flesch-Kincaid grade — all local. The "AI Rewrite" button is the one and only feature that sends your text to Google Gemini.

3. **Local-first hybrid** — input decoding is local, but the decoded content is automatically sent to an AI model for a downstream task. Example: the QR Code Scanner. The image is decoded locally; the resulting URL is auto-classified by the AI safety check.

The trick is that pattern 3 is more useful to users (no extra button click) but pattern 2 is more honest (opt-in per call). I use 3 only when the user's intent in scanning a QR code clearly includes "tell me if this is safe", and the data being sent is the URL — not anything personal.

---

## What Stays Local, By Default

For each AI feature, I drew an explicit list of inputs that must never leave the browser:

### Invoice / Quote / Receipt Generators

| Stays local | Sent to AI |
|---|---|
| Business name, address, tax ID | The 3–600 chars of "what did I do" notes you type into the AI Suggest dialog |
| Client details | (nothing else from the form) |
| Logo image | |
| All line item amounts, quantities, rates | |
| All tax math, currency, discount | |
| The generated PDF | |

The trust boundary here is the AI Suggest dialog. The user has to actively click a sparkle icon to invoke it. The dialog has a clear "Sent to Google Gemini. Don't paste confidential info" disclosure right above the input.

### QR Code Scanner

| Stays local | Sent to AI |
|---|---|
| The QR image / camera feed | The decoded URL string (only for URL-type QRs) |
| Plain text QR contents | The pre-flagged heuristic signals from local analysis |
| vCard contents | |
| WiFi credentials | |
| GPS coordinates | |

The trust boundary here is "is the decoded content a URL?". Plain text and vCard scans skip the AI entirely. Only URL scans go through the safety check — and even then, the image itself is never uploaded.

### URL Explainer

| Stays local | Sent to AI |
|---|---|
| URL parsing (protocol, host, path, params) | The URL itself, only when the AI portion runs |
| Safety flag detection (punycode, embedded creds, suspicious TLDs) | List of "unknown" parameters (ones not in the built-in 40+ list) |
| The known-parameter explanation list (UTM, fbclid, gclid, etc.) | |

The trust boundary here is "known vs unknown parameters". The tool ships with a list of 40+ commonly used parameters and their purposes. Only parameters NOT in that list get sent to the AI for explanation. Most URLs you paste will hit the known list entirely and never trigger an AI call.

---

## The Model-Choice Dimension

A second architectural decision: which model to use for which task. Google AI Studio's free tier gives different daily quotas to different models, and the quality bar differs by task.

| Feature | Model | RPD | Why |
|---|---|---|---|
| Word Counter Rewrite | gemini-3.1-flash-lite | 500 | Tone and voice matter; users notice bad prose |
| Invoice/Quote/Receipt line items | gemini-3.1-flash-lite | (shared with Rewrite) | Same reason — professional voice matters |
| QR Safety Check | gemma-3-27b-it | 1,500 | Classification task. "Safe / suspicious / dangerous" doesn't need prose nuance. |
| URL Explainer (unknown params) | gemma-3-27b-it | 1,500 | Structured template output ("this param means X"). No creative writing. |

Three observations from running this in production:

1. **Mixed-model strategy works.** Reserving Gemini for nuanced tasks and Gemma for classification means neither quota becomes the bottleneck. The Word Counter and Invoice tools share Gemini's 500 RPD. QR Safety and URL Explainer each get their own 1,500 RPD on Gemma. Total capacity across the four AI features is 3,500 model calls per day — plenty for organic traffic.

2. **The free-tier limit on Gemini 2.5 Flash is 20 RPD.** I learned this the hard way after launching with that model. Twenty requests per day is fine for a demo, useless for a live tool. Always check the actual quota at aistudio.google.com/apikey before committing to a model — the marketing pages list flagship caps; the free-tier numbers are buried.

3. **Local heuristics still do the heavy lifting.** In the QR Safety Check, the local heuristics produce a verdict (URL shortener detected, punycode domain, embedded credentials, etc.) before any AI call. The AI is asked to provide a second opinion and a plain-English summary — but the verdict is `max(local, AI)`, so the AI can only escalate the warning, never downgrade it. This means even if the AI fails or hits the quota, the heuristics still protect users.

---

## Server-Side Rate Limiting

Every AI route has an in-memory rate limit keyed by client IP:

```
const RATE_LIMIT_WINDOW_MS = 60_000;
const RATE_LIMIT_MAX = 10;  // (15 for safety checks)
const rateLimitBuckets = new Map<string, { count: number; resetAt: number }>();
```

This is the simplest possible rate-limit implementation — a Map that lives for the lifetime of the Node.js process. It's not perfect (resets on deploy, doesn't survive multi-instance) but it covers the realistic abuse case: one user hammering the AI route with a script.

For multi-instance protection I'd need Redis or a per-IP queue in MariaDB. The site runs on Hostinger with two Passenger Node.js instances behind LiteSpeed — so an attacker who hits both instances could double the rate. I accept this for now because:

- Hitting the per-IP limit on both instances simultaneously requires a deliberate distributed attack
- The daily AI quota is the actual safety net — a runaway script will hit Google's 1,500 RPD before it costs me anything
- Adding Redis to a static-site-shaped deployment is over-engineering for the current traffic

---

## Privacy Disclosure UX

The disclosure pattern I settled on:

1. **Inline above the AI action, not in a footer.** The text "Sent to Google Gemini — don't paste confidential info" sits immediately above the textarea in the AI Suggest dialog. It's not a separate "Privacy" page link. The user reads it as they decide to click.

2. **Cover-all phrasing at the tool level too.** On the Word Counter page, the About paragraph explicitly says "AI Rewrite is the only feature that sends data anywhere — and only when you click the Rewrite button." This sets expectations before the user even tries the feature.

3. **No tracking, no analytics on AI inputs.** The AI route doesn't log the request body. Google may log it on their end (it's their API), but I'm not adding a second layer of "we also kept a copy".

The general principle: **the user should never be surprised that data left their device**. Surprise is the failure mode that destroys trust; verbose-but-clear disclosure is the cost of avoiding it.

---

## Edge Cases I Got Wrong on the First Try

- **Empty AI responses.** The first version of the invoice description route returned a 502 when Gemini's response had `finishReason: SAFETY` and empty text. The fix was to detect empty `candidates[0].content.parts[0].text` and return a 502 with a clearer error message — "AI returned no usable options. Try adding more detail to your notes."

- **The CDN blocking large JSON POST bodies.** Hostinger's hCDN (the CDN in front of LiteSpeed) blocks `application/json` POST bodies larger than ~40 KB with a 403 in ~60ms — the request never reaches Node.js. None of the AI features hit this because the payloads are small, but it bit me on a separate PDF upload feature. Fix was switching to `multipart/form-data` with the JSON stringified into a field.

- **AI verdict escalation, not replacement.** My first QR Safety design used the AI's verdict directly when present. Then I realized the AI sometimes feels good about URLs that local heuristics correctly flag as suspicious (e.g. URLs through known shorteners). The fix was `max(local_verdict, ai_verdict)` — the AI can escalate but never downgrade.

- **The known-parameter list is more powerful than I expected.** When I started the URL Explainer, I assumed most URLs would need AI calls. After building the 40+ known-param list (UTM family, ad click IDs, OAuth flow params, affiliate tags), I realized 95% of URLs paste-tested were fully covered by the list. The AI is reserved for the genuinely-novel parameters — which is much rarer than I expected.

---

## What I'd Do Differently

If I were starting from scratch today:

1. **Build the heuristics layer first, always.** Every AI feature should have a "what can we figure out without calling the AI" pass that runs first. It saves quota, runs faster, and creates a fallback path when the AI service is down or rate-limited.

2. **Pick the model based on task type, not on which is newest.** Latest model ≠ best model for the job. Classification tasks with strict output shapes work better on Gemma (and use its higher quota). Nuanced text generation needs Gemini.

3. **Disclosure inline, not in policy pages.** The user-facing disclosure has to be where the user is about to act, in plain language, with a specific example of what gets sent.

4. **Treat the daily AI quota as the budget, not the rate limit.** Per-IP rate limits protect against abuse. The daily quota is the real spend control — design features so a hot day of organic traffic doesn't blow through the budget.

---

## Related Tools

- [free AI text rewriter that simplifies prose to any reading grade level](https://ultimatetools.io/tools/text-tools/word-counter/) — the original AI feature, same architecture pattern
- [QR code generator with WiFi, vCard, and URL support](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — counterpart to the AI-safety-checked scanner
- [free password generator with passphrase mode and strength checker](https://ultimatetools.io/tools/security-tools/password-generator/) — fully local example of the older architecture

---

The hard part of adding AI to a privacy-positioned product is not the AI code. It's deciding what data crosses the trust boundary, surfacing that decision to the user in a way they actually read, and resisting the temptation to make the AI invocation invisible just because it would be smoother UX. Smoother is exactly the wrong direction here. The whole point is to make it deliberate.

**[Try the AI features — Invoice line items, QR Safety, URL Explainer, Text Rewriter →](https://ultimatetools.io/tools/)**