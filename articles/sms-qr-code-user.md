# How to Create an SMS QR Code — Pre-Write the Message, Recipient Scans and Sends

An SMS QR code opens the native text messaging app on any smartphone with the recipient number and message already filled in. The user scans, reviews the message, taps send. One step instead of five.

Here's how to create one, free.

---

## What does an SMS QR code do?

When scanned, it opens the phone's default SMS app with:
- The recipient number pre-filled
- The message body pre-filled (optional — you can leave it blank)

The user taps Send. That's it.

The QR encodes an `SMSTO:` URI:

```
SMSTO:+15550100:Hi, I need help with my order
```

On iOS, the equivalent format is:

```
sms:+15550100&body=Hi, I need help with my order
```

Modern QR scanners on both Android and iOS handle this correctly — the standard QR generators output a format that works on both platforms.

---

## How to create an SMS QR code

Go to the [QR Code Generator at Ultimate Tools](https://ultimatetools.io/tools/misc-tools/qr-code-generator/).

Select the **SMS** content type from the tab bar.

Fill in:
- **Phone number** — include the country code (e.g. +1, +44, +91)
- **Message** (optional) — the pre-written text that appears in the message body

Customize style — colors, dot shape, corner style, logo overlay. Click **Generate QR Code**.

Download as PNG or SVG.

---

## Pre-written message or blank?

**Pre-written message:** The user scans, sees a message already composed, taps Send. Zero effort required. Best for support lines, feedback requests, and anything where you want a specific message format.

**Blank message:** The user scans, the app opens with your number ready, they type their own message. Good for general contact, where you can't predict what they'll say.

If you want pre-filled, keep the message short and specific. A long message looks alarming before the user sends it.

---

## When to use an SMS QR code

**Customer support** — put the QR on product packaging, receipts, or warranty cards. Pre-fill: "Hi, I need help with [product name]." Customer scans, message goes straight to support. No number hunting, no copy-paste.

**Event RSVPs** — "Scan to confirm attendance." Pre-fill: "I'll be attending [Event Name] on [Date]." One scan confirms in seconds. Useful for informal events where a form would be overkill.

**Feedback requests** — after a service or purchase. Pre-fill: "Quick question about my experience at [Business Name]..." Lowers the barrier to reaching out.

**Appointment reminders** — "Need to reschedule? Scan to message us." Pre-fill: "I'd like to reschedule my appointment."

**Restaurants and cafes** — "Scan to message us directly." Useful for noise-sensitive environments where calling isn't practical.

---

## SMS QR vs WhatsApp QR

An **SMS QR** uses the phone's native SMS app — works for every phone number, no third-party app required. Universal reach.

A **WhatsApp QR** opens WhatsApp to a specific contact. Requires both parties to have WhatsApp installed. But WhatsApp messages are free internationally and include delivery receipts.

Use SMS QR for maximum reach (every phone supports SMS). Use WhatsApp QR when your audience is known to use WhatsApp, or when international messaging costs matter.

---

## What the recipient sees

**Android:** The Messages app (or default SMS app) opens with the number and message pre-filled. One tap to send.

**iOS:** The Messages app opens with the number and message pre-filled. One tap to send.

Both platforms handle the SMS scheme natively — no app install required.

---

## Include the country code

`+1 555 0100` works everywhere. `555 0100` may not parse correctly on a phone registered to a different country code. Always include `+` and the country code — it has no downside for domestic recipients and ensures international recipients can use the QR too.

---

## Try it

**[Create your SMS QR code at Ultimate Tools](https://ultimatetools.io/tools/misc-tools/qr-code-generator/)** — free, no signup, download as PNG or SVG.

Other QR code types:
- [Phone Number QR Code](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — open the dialer with one scan
- [WhatsApp QR Code](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — open a WhatsApp chat directly
- [vCard QR Code](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — save a full contact card with one scan
- [Email QR Code](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — pre-fill an email compose window