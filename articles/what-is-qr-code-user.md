# What Is a QR Code and How Does It Work?

QR codes are everywhere — restaurant menus, packaging, event tickets, business cards. But most people have no idea how they actually store and transmit information.

Here's exactly how they work, from the grid of dots to the decoded URL.

---

## What "QR" Stands For

QR stands for **Quick Response**. The format was invented in 1994 by Denso Wave (a Toyota subsidiary) for tracking automotive parts through manufacturing. The goal: a code that could be scanned faster than a barcode and store far more data.

It worked. QR codes can hold up to 4,296 alphanumeric characters. A standard barcode holds around 20.

---

## What a QR Code Actually Is

A QR code is a **two-dimensional matrix of black and white squares** called modules. Unlike a barcode (which only encodes data horizontally), a QR code encodes data in both directions — which is why it stores so much more.

The pattern is not random. Every region has a specific role:

**Finder patterns** — the three square-within-square symbols in three corners. They tell the scanner "this is a QR code" and establish orientation. This is why QR codes always have those three identical corner squares.

**Timing patterns** — alternating black/white lines along the top and left edges. They let the scanner determine the size of each module.

**Alignment patterns** — smaller position markers used in larger QR codes to correct for distortion.

**Data modules** — the actual encoded content, spread across the remaining area.

**Error correction modules** — redundant data that lets scanners read the code even when part of it is damaged, dirty, or covered.

---

## How Data Is Encoded

QR codes don't store a URL as plain readable text embedded in a grid. The encoding process:

1. The input (URL, text, etc.) is converted to a binary sequence
2. Error correction bytes are calculated using Reed-Solomon codes and added
3. The full binary sequence is mapped to module positions on the grid
4. A **mask pattern** is applied — one of eight patterns that XOR with the data to prevent large solid areas of the same color (which confuse scanners)

The result is the distinctive black-and-white grid.

---

## Error Correction Levels

QR codes have four error correction levels:

| Level | Recovery capacity | Use case |
|-------|------------------|----------|
| L (Low) | ~7% | Clean, undamaged print |
| M (Medium) | ~15% | Most everyday use |
| Q (Quartile) | ~25% | Industrial, outdoor |
| H (High) | ~30% | Logo embedded in QR code |

Higher error correction = more modules needed = larger, denser QR code. This is why QR codes with logos in the center still scan — the logo sits in the error correction zone, and the missing data is reconstructed.

---

## What QR Codes Can Store

QR codes encode more than just URLs. Common content types:

- **URL** — opens a website (most common)
- **Plain text** — a message or instructions
- **Wi-Fi credentials** — SSID + password, for one-tap network connection
- **vCard** — contact details (name, phone, email, address)
- **Email** — opens a pre-addressed compose window
- **Phone number** — opens the dialler
- **Location** — coordinates or a maps link
- **Calendar event** — adds an event to the device calendar

The content type is determined by a prefix in the encoded data. For example, Wi-Fi QR codes start with `WIFI:S:NetworkName;T:WPA;P:password;;`. The phone's QR scanner reads the prefix and opens the appropriate app automatically.

---

## Static vs Dynamic QR Codes

**Static QR codes** encode the destination directly. If you want to change where the code points, you generate a new code and reprint it.

**Dynamic QR codes** encode a short redirect URL. The redirect target can be updated at any time without changing or reprinting the physical code. Most business and marketing QR codes are dynamic.

Dynamic codes also support scan tracking — you can see how many times a code was scanned, from where, and on what device.

---

## How to Generate and Scan QR Codes

To generate a QR code with custom style, error correction level, and content type:

**[QR Code Generator](https://ultimatetools.io/tools/misc-tools/qr-code-generator/)** — supports 10 content types (URL, Wi-Fi, vCard, text, and more), custom colors, dot shapes, and logo embedding.

To scan a QR code from a screenshot, image file, or your camera:

**[QR Code Scanner](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/)** — upload any image or use your device camera. Decodes instantly in the browser, nothing uploaded to a server.

Both are free, no login required.

---

## Summary

A QR code is a 2D matrix where:
- Three corner squares establish orientation
- Binary data is mapped to module positions
- Error correction allows partial damage recovery
- A mask pattern prevents scanning failures from solid regions

What looks like a random grid of dots is actually a precisely structured data format — one that your phone decodes in under a second.