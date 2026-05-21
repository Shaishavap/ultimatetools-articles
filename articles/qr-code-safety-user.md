# Is This QR Code Safe? How to Check What a QR Code Links to Before You Scan It

QR codes are everywhere — restaurant menus, event tickets, product packaging, parking meters, email attachments. Most are harmless. Some aren't. The problem: you can't tell the difference by looking at the code.

Here's how to check what a QR code links to before you follow it anywhere.

Inspect any QR code free: [QR Code Scanner — Upload Image or Use Camera](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/)

---

## Why QR Code Safety Matters

A QR code is just a link in disguise. Scanning one is no different from clicking a URL you've never seen — except with URLs, you can at least read the domain before clicking. With a QR code, you're flying blind.

**QR phishing (quishing)** is a real and growing attack vector. Common scenarios:

- Fake parking meter QR codes that redirect to payment card skimmers
- QR codes in phishing emails that bypass email link scanners (which check URLs, not images)
- Tampered QR code stickers placed over legitimate ones in public spaces
- QR codes in PDFs or screenshots shared in messages

The solution is simple: decode the QR code and read the URL before following it.

---

## How to Check a QR Code Before Scanning

### Method 1 — Upload the Image

If you have the QR code as an image file, screenshot, or photo:

1. Go to the [QR Code Scanner](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/)
2. Click **Upload Image** and select your file
3. The decoded content appears instantly — read it before clicking

This works for QR codes in emails, PDFs, screenshots, and downloaded images.

### Method 2 — Use Your Camera to Preview

If the QR code is on a physical object:

1. Open the [QR Code Scanner](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/)
2. Click **Use Camera** and point it at the QR code
3. The decoded URL appears on screen — review it before tapping to open

Your native phone camera often auto-opens URLs without showing them clearly. Using a browser-based scanner shows you the full URL first.

---

## What to Check in the Decoded URL

Once you have the URL, look for these red flags:

**Suspicious domain** — Does the domain match what you'd expect? A QR code on a McDonald's tray should link to mcdonalds.com, not mcdonalds-offers-2026.net.

**URL shorteners** — bit.ly, tinyurl.com, t.co hide the final destination. A QR code using a URL shortener is not necessarily malicious, but it's a reason to be cautious. Use a URL expander service to see where it leads before clicking.

**HTTP instead of HTTPS** — Legitimate payment and login pages always use HTTPS. An HTTP link asking for credentials is a warning sign.

**Unusual subdomains** — paypal.com is safe. paypal.secure-login.xyz is not. The actual domain is always the part immediately before .com/.org/.net.

**Misspellings** — arnaz0n.com, g00gle.com, paypall.com — common phishing tricks.

---

## QR Codes That Don't Contain URLs

Not every QR code is a link. The decoded content might be:

- **Plain text** — a message, WiFi password, or instructions
- **Contact card (vCard)** — name, phone, email — safe to read, decide before saving
- **Wi-Fi credentials** — network name and password — verify it matches the expected network name
- **Email or phone** — opens a compose window or dialler — safe to review

The scanner shows the raw content. Read it before taking any action.

---

## Is the Scanner Safe to Use?

Yes — the QR code scanner runs entirely in your browser. Your image is never uploaded to any server. The QR code image goes from your device to your screen without leaving your machine, using the [html5-qrcode](https://github.com/mebjas/html5-qrcode) library locally.

This means you can safely inspect QR codes from sensitive documents (internal company materials, private emails, financial documents) without the image being sent anywhere.

---

## Quick Reference — Safe vs Suspicious

| Signal | Safe | Suspicious |
|---|---|---|
| Domain | Matches expected brand | Random or misspelled |
| Protocol | HTTPS | HTTP for payment/login |
| URL length | Normal | Excessively long with random strings |
| Shortener | Direct URL | bit.ly hiding destination |
| Content type | URL, plain text, vCard | Asks for login immediately |

**Default rule:** If the URL looks unfamiliar, don't open it. Search for the business or service directly instead of following the QR code link.

---

Inspect any QR code before scanning — free, instant, no upload: [QR Code Scanner](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/)