# Quote → Invoice → Receipt: Full Freelance Billing Workflow Free

Three documents. Three stages. One workflow — built so each step flows into the next without re-typing your data.

## The Three-Document Billing Workflow

Every freelance or B2B transaction goes through three stages:

```
Quote (before work)  →  Invoice (after work)  →  Receipt (after payment)
QUO-001                  INV-001                   REC-001
```

Most tools handle only one of these. You end up copying client details from a quote into an invoice manually, then copying them again into a receipt. This tool covers all three — and bridges them with a single click.

## Stage 1: Quote

A quote establishes the price before work starts. Key fields that differ from an invoice:

- **Valid Until** instead of Due Date — protects you from price changes
- **QUO-001** numbering — visually distinct from invoices
- No binding payment obligation yet — it's a price proposal
- Notes field for scope, assumptions, and what's not included

[Quote Generator →](https://ultimatetools.io/tools/misc-tools/quote-generator/)

## Stage 2: Convert to Invoice — How the Bridge Works

When the client approves the quote, click **Convert to Invoice →**. Here's what happens under the hood:

```javascript
const handleConvertToInvoice = () => {
    // Map quote fields to InvoiceGenerator's persisted state format
    const invoiceData = {
        region: "Generic",
        template: state.template,
        senderName: state.senderName,
        senderAddress: state.senderAddress,
        senderEmail: state.senderEmail,
        clientName: state.clientName,
        clientAddress: state.clientAddress,
        taxRate: state.taxRate,
        taxLabel: state.taxLabel,
        customTaxLabel: state.taxLabel,
        customCurrency: sym, // e.g. "$", "£", "EUR"
        logoBase64: state.logoBase64,
        items: state.items.map(it => ({
            ...it,
            hsn: "",      // invoice has HSN codes (India GST) — blank for generic
            taxRate: "",  // per-item tax override — blank for generic
        })),
        notes: state.notes,
        terms: state.terms,
        // ... all other InvoiceGenerator fields set to safe defaults
    };

    // Write to the key InvoiceGenerator reads on mount
    localStorage.setItem("ult-invoice-saved", JSON.stringify(invoiceData));

    // Navigate — InvoiceGenerator pre-fills automatically
    router.push("/tools/misc-tools/invoice-generator/");
};
```

The key insight: **InvoiceGenerator already reads `ult-invoice-saved` from localStorage on mount** to restore a saved session. The quote tool writes to that same key before navigating. No API needed, no URL params, no re-typing.

What transfers: sender details, client details, all line items, tax rate and label, template style, logo, notes, terms, currency symbol.

What doesn't transfer: discount (not part of InvoiceGenerator's persisted schema) — re-enter if needed.

## Stage 2: Invoice

Once on the invoice page, everything is pre-filled. The Invoice Generator adds what quotes don't have:

- **Regional tax modes**: India (CGST+SGST/IGST), Canada (HST by province), UK (VAT), EU (VAT + reverse charge), Australia (GST + ABN), USA (sales tax)
- **Invoice number**: auto-increments from INV-001
- **Due Date**: 30 days out by default
- **Cloud sync**: Google Sign-In saves profile, clients, and invoice history across devices
- **UPI QR code**: India invoices embed a scannable UPI payment QR

Download the invoice PDF, send to client, wait for payment.

[Invoice Generator →](https://ultimatetools.io/tools/misc-tools/invoice-generator/)

## Stage 3: Receipt

Payment arrives. Now issue a receipt — proof that money was received. The Receipt Generator is linked from the Invoice Generator's related tools section.

Key differences from an invoice:

| | Invoice | Receipt |
|---|---|---|
| Timing | Before payment | After payment |
| Key label | Due Date | Date Paid |
| Header | INVOICE | RECEIPT + **PAID** badge |
| Extra field | — | Payment method + reference |

The PAID badge is drawn directly in the PDF using pdf-lib:

```javascript
// Green PAID badge in the receipt header (top-right)
const paidLabel = "PAID";
const paidPad = 5;
const paidW = fontBold.widthOfTextAtSize(paidLabel, 10) + paidPad * 2;
const paidH = 10 + paidPad * 2;

page.drawRectangle({
    x: width - 40 - paidW,
    y: headerY - 36,
    width: paidW,
    height: paidH,
    color: rgb(0.086, 0.529, 0.176), // green-700
});
page.drawText(paidLabel, {
    x: width - 40 - paidW + paidPad,
    y: headerY - 36 + paidPad,
    size: 10,
    font: fontBold,
    color: rgb(1, 1, 1),
});
```

Eight payment methods: Cash, Credit Card, Debit Card, Bank Transfer, UPI, Cheque, PayPal, Other. Add a reference number (transaction ID, cheque number, card last 4) for traceability.

[Receipt Generator →](https://ultimatetools.io/tools/misc-tools/receipt-generator/)

## How the Profile Persistence Works Across All Three Tools

Each tool has its own localStorage key for sender profile:

| Tool | Profile key | Sequence key |
|---|---|---|
| Quote Generator | `ult-quote-profile` | `ult-quote-seq` |
| Invoice Generator | `ult-invoice-saved` | `ult-invoice-last-number` |
| Receipt Generator | `ult-receipt-profile` | `ult-receipt-seq` |

They're intentionally separate. Your invoice profile may have a GSTIN or VAT number; your quote profile doesn't need it. The Convert to Invoice bridge is the only cross-tool write — and it writes to the invoice's own key, so nothing is overwritten permanently.

## Why localStorage Over a Database for This

- **No account friction** — most people who need a quick quote don't want to sign up
- **Instant** — no round-trip to a server
- **Private** — your client names and amounts never leave the browser (unless you sign in to Invoice Generator for cloud sync, which is opt-in)
- **Works offline** — open the tool on a plane, generate a quote, done

The tradeoff: clearing browser data loses your profile. But for sender details that rarely change (your name, address, logo), that's an acceptable trade for 99% of users.

## The Full Workflow in Practice

1. **New project**: open [Quote Generator](https://ultimatetools.io/tools/misc-tools/quote-generator/), add line items, set Valid Until 30 days, download QUO-001.pdf, send to client
2. **Client approves**: click Convert to Invoice → — pre-filled, adjust region/tax if needed, download INV-001.pdf, send
3. **Payment received**: open [Receipt Generator](https://ultimatetools.io/tools/misc-tools/receipt-generator/), fill client + items, select payment method, download REC-001.pdf, send as confirmation

Total re-typing: zero on steps 1→2 (bridge handles it). Minimal on step 3 (client name + items — ~30 seconds).

## Related Tools

- [Quote Generator](https://ultimatetools.io/tools/misc-tools/quote-generator/) — PDF quotes, Valid Until, 24 currencies, Convert to Invoice in one click
- [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) — Canada HST, UK VAT, Australia GST, EU reverse charge, India GST, USA sales tax
- [Receipt Generator](https://ultimatetools.io/tools/misc-tools/receipt-generator/) — PAID stamp, 8 payment methods, transaction reference
- [Currency Converter](https://ultimatetools.io/tools/misc-tools/currency-converter/) — live ECB rates before quoting international clients

---

Full billing workflow — Quote, Invoice, Receipt — all free, no account: [Ultimate Tools](https://ultimatetools.io/tools/misc-tools/)