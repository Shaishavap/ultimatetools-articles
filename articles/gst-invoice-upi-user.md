# How to Create a Free GST Invoice Online — With UPI QR Code, No Signup

If you're a freelancer or small business in India, you need GST invoices that:

- Show CGST + SGST (or IGST for inter-state) correctly
- Include your GSTIN and HSN/SAC codes
- Have a way for clients to pay you without you chasing them

Most free invoice tools handle the first two badly and ignore the third entirely. This post covers a browser-based tool that gets all three right — including a UPI QR code printed directly on the PDF.

---

## What the Tool Does

[UltimateTools Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) is a free, browser-based GST invoice maker. No account. No subscription. Your invoice data never leaves your device — the PDF generates entirely in your browser.

Key features for Indian invoices:

- **CGST + SGST split** — enter your GST rate once, the tool splits it (e.g. 18% → 9% CGST + 9% SGST shown as two separate lines)
- **IGST mode** — toggle for inter-state transactions (single IGST line instead of CGST+SGST)
- **GSTIN field** — shown in the sender block on the PDF
- **HSN/SAC codes** — per line item column on the invoice table
- **UPI QR code** — scannable QR in the bottom-right corner of the PDF, pre-loaded with your UPI ID and the exact invoice total

---

## Step-by-Step: Create a GST Invoice

### Step 1 — Open the tool and select India

Go to the [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/)

The India region (🇮🇳) is selected by default. The form shows Indian-specific fields: GSTIN, HSN/SAC, CGST/SGST toggle.

### Step 2 — Fill in your business details

In the **From** section:
- Business name
- Address
- GSTIN (e.g. `27AABCU9603R1ZX`)
- Email (optional)

### Step 3 — Fill in client details

In the **Bill To** section:
- Client name
- Client address
- Client GSTIN (optional — required if they want to claim ITC)

### Step 4 — Choose tax type

**CGST + SGST** — use this when you and your client are in the same state. The tool automatically splits the rate (18% → 9% CGST + 9% SGST).

**IGST** — use this when you and your client are in different states. Shows as a single IGST line at the full rate.

Enter your GST rate percentage. Common rates: 5%, 12%, 18%, 28%.

### Step 5 — Add line items

Each row has: Description | HSN/SAC code | Qty | Rate | Amount

The tool calculates the line total and applies tax to the subtotal. Supports multiple items.

### Step 6 — Add your UPI ID

There's a **UPI ID** field below the payment details section. Enter your UPI ID (format: `name@bankname` or `9876543210@upi`).

This is what gets encoded in the QR code on the PDF.

### Step 7 — Download

Click **Download Invoice**. The PDF opens for download — no upload, no server, instant.

---

## What the PDF Looks Like

The downloaded invoice includes:

- Your business name and GSTIN at the top
- Invoice number, date, and due date
- Client details
- Line items table with HSN/SAC codes
- Subtotal → CGST line → SGST line → Total
- Notes and payment terms sections
- **UPI QR code** in the bottom-right corner with "Scan to Pay (UPI)" label and your UPI ID printed below it

The QR encodes a UPI deep link: `upi://pay?pa=yourname@upi&am=12400.00&cu=INR`. Every UPI app (GPay, PhonePe, Paytm, BHIM, bank apps) opens the payment screen with your ID and the exact amount pre-filled when scanned.

---

## CGST vs SGST vs IGST — Quick Reference

| Transaction type | Tax type | Example at 18% |
|---|---|---|
| Seller and buyer in same state | CGST + SGST | 9% CGST + 9% SGST |
| Seller and buyer in different states | IGST | 18% IGST |
| Export outside India | Zero-rated (exempt) | 0% |

---

## Invoice Number Auto-Increment

The tool remembers your last invoice number in your browser's localStorage and suggests the next one automatically. You can override it at any time.

Format: `INV-001`, `INV-002`, etc. — or set your own like `FY26-001`.

---

## Download and Share

Once downloaded, the PDF is a standard file you can:

- Email as an attachment
- Share via WhatsApp (works on mobile)
- Print and hand over
- Save to Google Drive or Dropbox

The UPI QR works whether the client opens the PDF on their laptop (they point their phone camera at the screen) or on mobile (they can share it to another device or screenshot it).

---

## Free, Always

No signup. No credit card. No watermark. No "free plan" with limits.

Create your first GST invoice with UPI QR: [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/)