# QR Codes Can't Be Trusted by Sight — Here's a Free AI Safety Check That Reads Them For You

A QR code is a 1-inch square of dots that, when scanned, can do anything from open a menu to drain a wallet. The square itself carries no visible clue about what it actually links to. You scan it, your phone opens the URL, and by the time you notice the address bar looks wrong, your one-time login code has already been requested.

There's now a [free QR code scanner with an AI safety check](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/) — every URL you scan is automatically classified Safe, Caution, Suspicious, or Dangerous before you open it. Image decoding runs locally in your browser; the safety verdict combines local heuristics with a Google Gemma classification pass.

---

## What the Safety Check Looks For

When you scan a QR code, the decoded content is automatically analysed by a layered system. The local heuristics fire first — they catch the obvious cases instantly without any network call.

The heuristics check for:

- **URL shorteners** (bit.ly, t.co, tinyurl, etc.) — the actual destination is hidden until you click. Default: Caution.
- **Suspicious TLDs** (.tk, .ml, .ga, .xyz, .top and others commonly abused for short-lived scams).
- **IP-literal hosts** (`http://203.0.113.42/login`) — almost never used by legitimate businesses.
- **Punycode domains** (`xn--something.com`) — a way to encode internationalized domain names that's frequently abused to make `аpple.com` look like `apple.com` (the first uses a Cyrillic 'а', the second is Latin 'a').
- **Embedded credentials** (`https://user:pass@example.com/`) — almost always malicious.
- **Phishing-adjacent keywords** in the URL ("verify", "secure", "login", "wallet", "airdrop", "claim", "prize"). One match → minor flag; three matches → escalates to Suspicious.
- **Excessive subdomain depth** (`login.secure.update.account.bank.evil.com`) — a classic phishing pattern.
- **Missing HTTPS** — flagged.

A numerical risk score (0–100) is built from these signals. The local heuristics alone produce a verdict: 0–14 Safe, 15–34 Caution, 35–59 Suspicious, 60+ Dangerous.

---

## Then the AI Gets a Second Opinion

After the heuristics finish, the URL plus the list of pre-flagged signals is sent to Google Gemma (the `gemma-3-27b-it` model). Gemma is asked to produce two lines:

1. A verdict (safe / caution / suspicious / dangerous)
2. A single plain sentence explaining why, written for a non-technical user

The final verdict is the **maximum** of the local and the AI verdict — the AI can only escalate, never downgrade. This is intentional. A local heuristic flag like "uses a URL shortener" should never be overridden by an AI feeling good about the link. But if the heuristics miss something the AI catches (a brand-mimicking domain that the punycode check didn't flag, for example), the AI can upgrade the warning level.

The plain-English sentence is the part most users actually read. "Several traits match common scam patterns — the destination is hidden behind a shortener and the path mentions login. Do not enter your password here." is more useful than a raw score of 72.

---

## What Changes When a Scan Is Flagged Dangerous

The result panel is colour-coded:

- **Safe** → emerald, normal "Open URL" button.
- **Caution** → yellow, normal Open URL button + a footer hint to verify before entering passwords.
- **Suspicious** → orange, Open URL button still works but the panel adds a clear "do not enter passwords or payment info" warning.
- **Dangerous** → red, **the Open URL button is replaced** with a red "Open Anyway" button that triggers a JavaScript confirm dialog before opening. The user has to actively confirm twice.

The point isn't to make Dangerous links impossible to open — sometimes the user really does need to inspect a phishing site for a report. The point is to break the muscle-memory tap that gets people scammed.

---

## Why the Heuristics Run First (and Locally)

Plain text and vCard contents never get sent to the AI — those are decoded entirely in the browser and skip the safety check. Only URL and payment-link content go to Gemma, and only after the local heuristics have done their pass.

Three reasons for this design:

1. **Privacy default.** The fewer scans that leave the browser, the better. A plain-text QR code containing "Happy Birthday Sarah" has no reason to be sent to any server.
2. **Latency.** Heuristics return in under 5ms. The AI call takes 200–800ms. Showing the heuristic result instantly and then letting the AI refine it feels faster than waiting for one round-trip.
3. **Cost / rate-limit headroom.** The AI has a daily quota. Skipping AI on cases where local heuristics already produce a high-confidence answer keeps the quota available for borderline cases where the AI actually helps.

---

## What the Safety Check Doesn't Do

It does **not**:

- Follow the URL to see the final destination after redirects. Following a suspicious URL server-side would expose the server's IP and any cookies the URL sets — exactly the bait phishing sites are testing for. The tool analyses the URL you scanned, not the page it leads to.
- Check the URL against a real-time threat-intelligence feed. The heuristics are pattern-based, not reputation-based. A URL that's brand-new and has no known reputation will still get checked for structural red flags, but a known phishing URL won't necessarily ring louder because we don't query VirusTotal-style services.
- Replace your own judgment. If the verdict is Caution and the link is from someone you trust, open it. If the verdict is Safe but the link came from an unsolicited message, still be careful. The verdict is one signal — context is yours.

---

## Use Cases

- **Restaurant menu QR codes** — usually safe, but check anyway. Stickers can be physically replaced by attackers in high-traffic locations.
- **Parking meter / EV charger payment QR codes** — a known attack vector. Replaced stickers redirect to fake payment pages that capture card details.
- **Event check-in QR codes from email** — verify the link goes to the expected event-platform domain, not a lookalike.
- **Anything received via WhatsApp or text message** — these messages are often used for phishing, and QR codes in screenshots are common.
- **Crypto / wallet QR codes** — payment QRs are flagged with "verify recipient before sending" regardless of other signals.
- **Random posters and stickers** — the AI Safety Check is the difference between scanning out of curiosity and getting redirected to a credential-harvesting site.

---

## Privacy and Cost

- Image decoding: **local**. Your image / camera feed never leaves your device.
- Plain text and vCard scans: **local only**. No AI call.
- URL / payment-link scans: the URL is sent to Google Gemma for the safety classification step. Heuristic flags are included so Gemma sees them. The QR image itself is never uploaded.
- Free. Per-IP rate limit is 15 scans per minute — well above normal use. Daily quota is shared and resets daily.

---

## Related Tools

- [QR code generator with WiFi, vCard, and URL support](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — generate your own QR codes for menus, business cards, or WiFi
- [URL explainer that decodes UTM tags and flags suspicious links](https://ultimatetools.io/tools/security-tools/url-explainer/) — paste any URL and get a parameter-by-parameter breakdown with safety flags
- [free password generator with passphrase mode and strength checker](https://ultimatetools.io/tools/security-tools/password-generator/) — for the password you should generate fresh after almost-falling-for a phishing QR

---

The reason QR-code phishing works is that humans treat the visual square as a black box. You wouldn't click a random unmarked link in an email — but you'll scan a sticker on a parking meter because the action of scanning feels neutral. The AI Safety Check inserts a step between the scan and the click, just long enough to let you reconsider.

**[Try the free QR code scanner with AI safety check →](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/)**