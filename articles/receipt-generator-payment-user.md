# Free Receipt Generator Online — PDF Receipts with PAID Stamp

An invoice asks for payment. A receipt confirms it arrived. This tool handles the post-payment step — generating a clean PDF receipt with a PAID badge, payment method, optional reference number, and your logo. No account, no watermark, runs in the browser.

## Receipt vs Invoice — The Key Difference

Both documents have your name, the client's name, and a list of items with amounts. The difference is timing and purpose:

| | Invoice | Receipt |
|---|---|---|
| Issued | Before payment | After payment |
| Header | INVOICE | RECEIPT |
| Key date | Due Date | Date Paid |
| Status indicator | — | **PAID** badge |
| Extra field | Payment terms | Payment method + reference |
| Purpose | Requesting payment | Confirming payment received |

Sending an invoice after payment (instead of a receipt) creates confusion — the client thinks you're asking for money you already have.

## When You Need a Receipt

- **Cash transactions** — no bank record exists; a receipt is the only proof
- **Client expense claims** — many companies require a vendor receipt before reimbursing employees
- **Tax records** — receipts prove that reported income was actually received
- **International payments** — bank transfers with a reference number + receipt = clear paper trail
- **Contractors and freelancers** — professionalism signals that you run a real business, not a side gig

## The PAID Badge — How It's Drawn in the PDF

The PAID badge appears in the top-right of the receipt header. It's drawn using [pdf-lib](https://pdf-lib.js.org/) — a filled green rectangle with white bold text:

```javascript
const paidLabel = "PAID";
const paidFontSize = 10;
const paidPad = 5;

// Calculate badge dimensions from text width
const paidW = fontBold.widthOfTextAtSize(paidLabel, paidFontSize) + paidPad * 2;
const paidH = paidFontSize + paidPad * 2;

// Position: right-aligned in header, below receipt number
const paidX = width - 40 - paidW;
const paidY = headerY - 36;

// Draw green rectangle
page.drawRectangle({
    x: paidX,
    y: paidY,
    width: paidW,
    height: paidH,
    color: rgb(0.086, 0.529, 0.176), // green-700
});

// Draw white PAID text inside
page.drawText(paidLabel, {
    x: paidX + paidPad,
    y: paidY + paidPad,
    size: paidFontSize,
    font: fontBold,
    color: rgb(1, 1, 1),
});
```

The badge is calculated dynamically from the text width, so it scales correctly regardless of font metrics. The live preview in the browser also shows the badge as a Tailwind `bg-green-600` span — same visual, different rendering.

## 8 Payment Methods

The receipt includes a "Payment Method" section at the bottom of the PDF:

| Method | When to use |
|---|---|
| **Cash** | In-person transactions — no bank record exists |
| **Credit Card** | Card terminal or payment link |
| **Debit Card** | Direct bank card payment |
| **Bank Transfer** | Wire/NEFT/SWIFT — add the reference ID |
| **UPI** | India instant payments — add UTR number |
| **Cheque** | Add cheque number for reference |
| **PayPal** | Online payment — add transaction ID |
| **Other** | Crypto, barter, platform-specific |

The optional **Reference** field accepts any string — transaction ID, cheque number, card last 4 digits. It appears in the PDF as `Paid via Bank Transfer | Ref: TXN-4829301`.

## Tax on Receipts

A receipt can include tax when:
- The client needs the tax amount for their own VAT/GST input credit claim
- You're issuing a receipt for a sale that included tax (e.g., retail)
- Local regulations require tax to appear on proof-of-payment documents

The tool has a configurable Tax Label (VAT / GST / Sales Tax / anything) and percentage. Tax is calculated on the subtotal and shown as a separate line before the total, labeled "Total Paid."

If you already showed tax on the invoice, you can set tax to 0% on the receipt — the receipt just confirms the total amount already agreed.

## Profile Persistence

Your sender details are saved to `localStorage["ult-receipt-profile"]` after every change. On the next visit, they pre-fill:

- Business name, address, email
- Logo (base64 encoded PNG/JPEG)
- Default template preference
- Currency
- Tax label and rate
- Default payment method

Only the client section and line items stay blank — those are unique to each receipt. Click **+ New Receipt** to increment the sequence number (REC-002, REC-003...) and clear client/items while keeping your profile.

## All 24 Currencies

USD, EUR, GBP, CAD, AUD, INR, JPY, CHF, SGD, AED, NZD, HKD, MXN, BRL, ZAR, SEK, NOK, DKK, PLN, KRW, CNY, IDR, MYR, THB.

All symbols are PDF-safe (ASCII equivalents for multi-byte characters) so Helvetica WinAnsi encoding doesn't produce garbled output. EUR renders as `EUR`, ₹ renders as `Rs.`, ₩ renders as `KRW`.

## Three PDF Templates

- **Simple**: clean white layout, zinc borders, no color
- **Professional**: indigo header band with white RECEIPT title — same style as the invoice
- **Minimal**: thin separator lines only, very clean

All three embed your logo (PNG or JPEG), the PAID badge, and the payment method block — the only difference is the visual style of the header.

## Where This Fits in the Billing Workflow

```
Quote (before work)  →  Invoice (after work)  →  Receipt (after payment)
QUO-001                  INV-001                   REC-001
```

The receipt is the final document in the workflow. Once issued, the transaction is complete and documented on both sides.

Related generators for the complete workflow:
- [Quote Generator](https://ultimatetools.io/tools/misc-tools/quote-generator/) — send a price proposal before starting
- [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) — request payment with regional tax (HST/VAT/GST/GST+ABN)
- [Receipt Generator](https://ultimatetools.io/tools/misc-tools/receipt-generator/) — confirm payment with PAID stamp and method

---

Issue a professional payment receipt with PAID stamp in seconds: [Receipt Generator](https://ultimatetools.io/tools/misc-tools/receipt-generator/)