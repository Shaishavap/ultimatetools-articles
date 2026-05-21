# How to Create a vCard QR Code — Save a Full Contact Card to Someone's Phone With One Scan

A vCard QR code opens an "Add Contact" prompt on any smartphone when scanned — name, phone number, email, company, and website all pre-filled. One scan, contact saved.

Here's how to create one in under a minute, free.

---

## What is a vCard QR code?

When a QR code encodes a vCard, scanning it triggers the phone's native contact import flow. The full contact details you encoded — name, job title, phone, email, company, website — appear in the Add Contact screen with one tap to save.

The QR encodes a standard `VCARD` format string:

```
BEGIN:VCARD
VERSION:3.0
FN:Jane Smith
ORG:Acme Corp
TEL:+1-555-0100
EMAIL:jane@acmecorp.com
URL:https://acmecorp.com
END:VCARD
```

Every modern Android and iOS device recognizes this format natively — no special app required.

---

## How to create a vCard QR code

Go to the [QR Code Generator at Ultimate Tools](https://ultimatetools.io/tools/misc-tools/qr-code-generator/).

Select the **vCard** content type from the tab bar.

Fill in your details:
- **Name** — full name as you want it saved
- **Phone** — include country code for international contacts (e.g. +91 for India, +1 for US/Canada)
- **Email** — work or personal
- **Company** — optional but recommended for business cards
- **Job Title** — optional
- **Website** — your personal site or company URL

Customize the style — dot shape, corner style, colors, logo overlay. Click **Generate QR Code**.

Download as PNG or SVG. Done.

---

## When to use a vCard QR code

**Business cards** — a QR in the corner of your card saves the contact instantly. No manual typing. No "let me find you on LinkedIn." They scan, they save, they have your number.

**Conference badges** — attendees scan your badge rather than photographing your card or exchanging details over email later.

**Email signatures** — embed the QR image in your email signature. Anyone reading your email on their phone can scan and save your number without switching apps.

**Product packaging and inserts** — "Scan to save our support number" is more frictionless than asking customers to type it in.

**Networking events** — put the QR on a tent card at your seat or a sticker on your laptop.

---

## vCard vs plain phone number QR

A plain phone number QR (`tel:+15550100`) opens the dialer with the number pre-filled. The contact isn't saved — the person has to manually create a contact afterwards.

A vCard QR saves the full contact — name, company, email, website — in one step. For professional networking, vCard is always the better choice.

---

## What gets saved

When someone scans your vCard QR:

- **Android:** Google Contacts opens with all fields pre-filled. One tap to save.
- **iOS:** The Contacts app opens an "Add Contact" screen. One tap to save.

The saved contact includes everything you encoded — not just the number. If they search your name later, your email and company come up too.

---

## Include your country code

Phone numbers without a country code may not import correctly on international phones. `+91 99999 00000` will always work. `99999 00000` might not parse correctly if the recipient's phone is set to a different region.

---

## Dynamic vs static vCard QR codes

A static vCard QR encodes your contact details directly. If your phone number or email changes, you need to generate a new QR and reprint.

A dynamic QR code redirects through a short URL. You can update the contact details behind it without reprinting — useful for business cards printed in large quantities, or for roles where the phone number might change.

The [Dynamic QR Code Generator at Ultimate Tools](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) supports this.

---

## Try it

**[Create your vCard QR code at Ultimate Tools](https://ultimatetools.io/tools/misc-tools/qr-code-generator/)** — free, no signup, download as PNG or SVG.

Other QR code types:
- [WhatsApp QR Code](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — open a WhatsApp chat directly
- [Email QR Code](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — pre-fill an email compose window
- [Wi-Fi QR Code](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — share network credentials without typing
- [Location QR Code](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — open Google Maps to an address