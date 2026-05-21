# Embedding UPI QR Codes in PDF Invoices with qr-code-styling and pdf-lib in Next.js

## The Problem

Our [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) already used **pdf-lib** to generate PDFs client-side and **qr-code-styling** for QR generation elsewhere in the app. Adding a UPI payment QR to Indian invoices seemed straightforward — but there's one non-obvious step: `qr-code-styling` produces a `Blob`, and `pdf-lib` needs a `Uint8Array`. You can't just pass one to the other.

This post covers the exact pipeline: UPI deep link → QR PNG blob → `ArrayBuffer` → `Uint8Array` → pdf-lib `embedPng` → page render.

---

## The UPI Deep Link Format

Every UPI app in India (GPay, PhonePe, Paytm, BHIM, all bank apps) understands a standard deep link format:

```
upi://pay?pa=<upi-id>&am=<amount>&cu=INR
```

- `pa` = payee address (your UPI ID: `name@bank`)
- `am` = amount in rupees, two decimal places
- `cu` = currency code (always `INR`)

When a UPI app scans a QR encoding this URL, it opens the payment confirmation screen with all fields pre-filled. The user just confirms with PIN or biometric.

```typescript
const upiUrl = `upi://pay?pa=${encodeURIComponent(upiId)}&am=${total.toFixed(2)}&cu=INR`;
```

`encodeURIComponent` on the UPI ID handles edge cases like `+` signs in some UPI IDs.

---

## Generating the QR as a PNG Blob

`qr-code-styling` is a browser-safe QR library with styling options. It's already in the project for other QR features, so no extra install. Key method: `getRawData('png')` returns a `Promise<Blob | null>`.

```typescript
const QRCodeStyling = (await import("qr-code-styling")).default;

const qr = new QRCodeStyling({
  width: 150,
  height: 150,
  data: upiUrl,
  dotsOptions: { type: "square", color: "#000000" },
  backgroundOptions: { color: "#ffffff" },
  qrOptions: { errorCorrectionLevel: "M" },
});

const qrBlob = await qr.getRawData("png");
```

We use a dynamic import (`await import(...)`) because this runs inside an async PDF generation function. The library is only loaded when an India invoice with a UPI ID is actually downloaded — no cost for other regions.

`errorCorrectionLevel: "M"` (15% recovery) is the right balance for UPI QRs. The `upi://` URL is short enough that even "M" keeps the QR simple and scannable at small sizes.

---

## The Bridge: Blob → Uint8Array

This is the step that isn't obvious. `pdf-lib`'s `embedPng` needs a `Uint8Array`, not a `Blob`. The path is:

```
Blob → ArrayBuffer (via .arrayBuffer()) → Uint8Array
```

```typescript
if (qrBlob) {
  const arrayBuffer = await (qrBlob as Blob).arrayBuffer();
  const pngBytes = new Uint8Array(arrayBuffer);
  const qrImage = await pdfDoc.embedPng(pngBytes);
  // ...
}
```

The `as Blob` cast is needed because TypeScript types `getRawData` return as `Blob | null | undefined` depending on the version — the null guard above already handles the null case.

---

## Positioning the QR on the PDF Page

Our A4 page is 595×842 points. We fix the QR in the **bottom-right corner** so it never overlaps the invoice content regardless of how many line items are on the page.

```typescript
const qrSize = 90;          // 90pt ≈ 32mm — scannable at that size
const qrX = width - 40 - qrSize;   // 40pt right margin
const qrY = 36;             // 36pt from bottom

page.drawImage(qrImage, {
  x: qrX,
  y: qrY,
  width: qrSize,
  height: qrSize,
});
```

In pdf-lib, `y=0` is the **bottom** of the page (not the top). So `y: 36` places the QR near the bottom edge.

Then add the label above and the UPI ID below:

```typescript
// Label above
const scanLabel = "Scan to Pay (UPI)";
const scanW = font.widthOfTextAtSize(scanLabel, 7);
page.drawText(scanLabel, {
  x: qrX + (qrSize - scanW) / 2,
  y: qrY + qrSize + 4,
  size: 7,
  font,
  color: rgb(0.4, 0.4, 0.4),
});

// UPI ID below
const upiLabel = upiId.slice(0, 32); // truncate long IDs
const upiW = font.widthOfTextAtSize(upiLabel, 6);
page.drawText(upiLabel, {
  x: qrX + (qrSize - upiW) / 2,
  y: qrY - 10,
  size: 6,
  font,
  color: rgb(0.4, 0.4, 0.4),
});
```

`(qrSize - textWidth) / 2` centers the text under/over the QR. We use 7pt for the label (readable but unobtrusive) and 6pt for the ID (secondary info).

---

## Wrapping in Error Handling

QR generation involves async operations that could fail (library load error, blob generation error, embedPng error). Wrapping in try/catch means a QR failure never blocks the PDF download:

```typescript
if (state.region === "India" && state.upiId) {
  try {
    // ... full QR generation and embedding pipeline
  } catch {
    // skip QR silently — PDF still downloads without it
  }
}
```

---

## Full Snippet

```typescript
// Inside your async generatePDF function, before pdfDoc.save()

if (state.region === "India" && state.upiId) {
  try {
    const QRCodeStyling = (await import("qr-code-styling")).default;
    const upiUrl = `upi://pay?pa=${encodeURIComponent(state.upiId)}&am=${total.toFixed(2)}&cu=INR`;

    const qr = new QRCodeStyling({
      width: 150, height: 150,
      data: upiUrl,
      dotsOptions: { type: "square", color: "#000000" },
      backgroundOptions: { color: "#ffffff" },
      qrOptions: { errorCorrectionLevel: "M" },
    });

    const qrBlob = await qr.getRawData("png");
    if (qrBlob) {
      const arrayBuffer = await (qrBlob as Blob).arrayBuffer();
      const qrImage = await pdfDoc.embedPng(new Uint8Array(arrayBuffer));

      const qrSize = 90;
      const qrX = width - 40 - qrSize;
      const qrY = 36;

      page.drawImage(qrImage, { x: qrX, y: qrY, width: qrSize, height: qrSize });

      const scanLabel = "Scan to Pay (UPI)";
      const scanW = font.widthOfTextAtSize(scanLabel, 7);
      page.drawText(scanLabel, {
        x: qrX + (qrSize - scanW) / 2, y: qrY + qrSize + 4,
        size: 7, font, color: rgb(0.4, 0.4, 0.4),
      });

      const upiLabel = state.upiId.slice(0, 32);
      const upiW = font.widthOfTextAtSize(upiLabel, 6);
      page.drawText(upiLabel, {
        x: qrX + (qrSize - upiW) / 2, y: qrY - 10,
        size: 6, font, color: rgb(0.4, 0.4, 0.4),
      });
    }
  } catch { /* skip QR on error */ }
}

const pdfBytes = await pdfDoc.save();
```

---

## Live Example

The full invoice generator (India region with UPI QR, plus Canada HST, Australia GST, UK VAT, EU VAT, USA sales tax) is live at the [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) — free, no signup, browser-only.