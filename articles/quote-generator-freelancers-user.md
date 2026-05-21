# Free Quote Generator Online — PDF Estimates for Freelancers

Before you invoice a client, you quote them. Before the quote, you estimate. This tool handles the pre-invoice stage — generating a clean PDF quote with a Valid Until date, auto-incrementing numbers, 24 currencies, and your logo embedded. No account needed, no watermark.

## Quote vs Invoice — Why the Distinction Matters

A **quote** (also called an estimate or proposal) is sent *before* work begins. It says: "here's what I'll charge, and this price is valid until [date]."

An **invoice** is sent *after* work is done. It says: "here's what you owe me, please pay by [due date]."

Sending an invoice before a quote was agreed upon looks unprofessional. Sending a quote after you've already done the work means you had no price agreement and could get underpaid.

| | Quote | Invoice |
|---|---|---|
| Sent | Before work starts | After work is done |
| Key date | Valid Until | Due Date |
| Purpose | Price agreement | Payment request |
| Number format | QUO-001 | INV-001 |
| Binding? | Not yet | Yes |

## When to Send a Quote

- **New client, fixed project** — define scope + price upfront
- **Variable scope work** — quote a range with assumptions noted
- **International billing** — lock in a price in the client's currency before exchange rates shift
- **Regulated industries** — contractors, consultants, and agencies often legally require a quote before work
- **Large amounts** — anything where a surprise invoice would damage the relationship

## How the Quote Number Works

Quote numbers auto-increment each time the page loads: QUO-001, QUO-002, QUO-003. The sequence is stored in localStorage and persists across sessions:

```javascript
const QUOTE_SEQ_KEY = "ult-quote-seq";

function getNextQuoteNumber(): string {
    const last = localStorage.getItem(QUOTE_SEQ_KEY);
    const n = last ? parseInt(last) + 1 : 1;
    localStorage.setItem(QUOTE_SEQ_KEY, String(n));
    return `QUO-${String(n).padStart(3, "0")}`;
}
```

You can also edit the number manually — useful if you're starting mid-sequence or use a different format like `Q-2024-047`.

## Your Profile Saves Automatically

Freelancers send a lot of quotes. Re-entering your name, address, email, logo, currency, and tax rate every time would be painful. The tool saves your sender profile to localStorage after every change:

```javascript
useEffect(() => {
    if (!profileLoaded) return;
    localStorage.setItem("ult-quote-profile", JSON.stringify({
        template: state.template,
        currency: state.currency,
        senderName: state.senderName,
        senderAddress: state.senderAddress,
        senderEmail: state.senderEmail,
        senderPhone: state.senderPhone,
        logoBase64: state.logoBase64,
        taxRate: state.taxRate,
        taxLabel: state.taxLabel,
        terms: state.terms,
    }));
}, [state.template, state.currency, state.senderName, /* ... */]);
```

On the next visit, your details pre-fill. The client section and line items stay blank — only your business profile is remembered. Click **+ New Quote** to generate the next quote number and clear client fields while keeping your profile intact.

## The PDF — What Goes In

The PDF is generated entirely in the browser using [pdf-lib](https://pdf-lib.js.org/) — no server, no upload. Here's what appears on every quote:

- **Header**: QUOTE title (centered), quote number + issue date + Valid Until (right-aligned), logo (left-aligned)
- **Sender / Quote For**: two-column layout — your details left, client right
- **Line items table**: Description, Qty, Rate, Amount — right-aligned numeric columns
- **Totals**: Subtotal → Discount → Tax → Total
- **Notes + Terms**: project scope, revision limits, payment schedule

Three templates change the visual only — the data is identical:
- **Simple**: clean white, zinc borders
- **Professional**: indigo header band with white title
- **Minimal**: thin separator lines, no color blocks

## Valid Until — Why This Field Matters

The Valid Until date (default: 30 days from today) protects you from two things:

1. **Inflation and subcontractor rate changes** — a quote accepted 6 months later shouldn't bind you to old pricing
2. **Scope creep by delay** — clients who sit on quotes too long often try to add to scope without renegotiating

Put it on every quote. If a client comes back after expiry, requote with current rates.

## 24 Supported Currencies

USD, EUR, GBP, CAD, AUD, INR, JPY, CHF, SGD, AED, NZD, HKD, MXN, BRL, ZAR, SEK, NOK, DKK, PLN, KRW, CNY, IDR, MYR, THB.

The currency symbol appears on every line item and in the totals. All symbols are ASCII-safe for PDF embedding (Helvetica WinAnsi encoding) — no garbled characters.

## After the Quote Is Accepted

Click **Convert to Invoice →** on the quote page. The tool maps your quote data (sender, client, items, tax, template, logo) into the Invoice Generator's localStorage format and navigates there automatically. The Invoice Generator pre-fills on load — no re-typing.

From there: download the invoice, get paid, then issue a receipt. Full workflow covered.

## Related Tools

- [Quote Generator](https://ultimatetools.io/tools/misc-tools/quote-generator/) — send a PDF quote before the invoice, Valid Until date, 24 currencies
- [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) — Canada (HST), UK (VAT), Australia (GST/ABN), EU, USA, India — cloud sync
- [Receipt Generator](https://ultimatetools.io/tools/misc-tools/receipt-generator/) — PAID stamp PDF after payment, 8 payment methods
- [Currency Converter](https://ultimatetools.io/tools/misc-tools/currency-converter/) — live ECB rates before quoting international clients

---

Send professional PDF quotes before every project: [Quote Generator](https://ultimatetools.io/tools/misc-tools/quote-generator/)