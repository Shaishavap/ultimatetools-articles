# Building a Multi-Region Tax Invoice Generator with pdf-lib in Next.js (7 Tax Systems, All Browser-Side)

Every invoice generator I found either locked multi-region tax behind a paywall or expected the user to know their own tax rates. I wanted a tool that handles India's CGST+SGST split, Canada's province-by-province HST, EU reverse charge, and Australian ABN — all client-side, no server, no upload.

Here's how I built it with `pdf-lib`, React state, and localStorage persistence.

---

## The Region System

Seven regions, each with different tax logic:

| Region | Tax type | Special fields |
|--------|----------|----------------|
| India | CGST+SGST (intra) or IGST (inter) | GSTIN, HSN/SAC code |
| Canada | HST/GST/PST by province | Business Number |
| UK | VAT 20% | VAT Number |
| EU | VAT % + reverse charge toggle | EU VAT Number |
| Australia | GST 10% | ABN |
| USA | Sales Tax % (optional) | EIN |
| Generic | Custom label + % | — |

The region config drives everything — currency symbol, default tax rate, tax ID label:

```typescript
const REGION_CONFIG: Record<Region, {
  currency: string;
  taxLabel: string;
  taxRate: number;
  taxIdLabel: string;
  currencySymbol: string;
}> = {
  India:     { currency: "INR", taxLabel: "CGST+SGST", taxRate: 18, taxIdLabel: "GSTIN", currencySymbol: "₹" },
  Canada:    { currency: "CAD", taxLabel: "HST",        taxRate: 13, taxIdLabel: "Business Number (BN)", currencySymbol: "C$" },
  UK:        { currency: "GBP", taxLabel: "VAT",        taxRate: 20, taxIdLabel: "VAT Number", currencySymbol: "£" },
  EU:        { currency: "EUR", taxLabel: "VAT",        taxRate: 21, taxIdLabel: "EU VAT Number", currencySymbol: "€" },
  Australia: { currency: "AUD", taxLabel: "GST",        taxRate: 10, taxIdLabel: "ABN", currencySymbol: "A$" },
  USA:       { currency: "USD", taxLabel: "Sales Tax",  taxRate: 0,  taxIdLabel: "EIN (optional)", currencySymbol: "$" },
  Generic:   { currency: "USD", taxLabel: "Tax",        taxRate: 0,  taxIdLabel: "Tax ID (optional)", currencySymbol: "$" },
};
```

---

## Canada: Province Lookup Table

Canada is the most complex — 10 provinces, four different tax structures (HST, GST-only, GST+PST, GST+QST):

```typescript
const CANADA_TAX: Record<string, { label: string; rate: number }> = {
  ON:  { label: "HST",      rate: 13 },
  NS:  { label: "HST",      rate: 15 },
  NB:  { label: "HST",      rate: 15 },
  NL:  { label: "HST",      rate: 15 },
  PEI: { label: "HST",      rate: 15 },
  BC:  { label: "GST+PST",  rate: 12 },
  AB:  { label: "GST",      rate: 5  },
  SK:  { label: "GST+PST",  rate: 11 },
  MB:  { label: "GST+RST",  rate: 12 },
  QC:  { label: "GST+QST",  rate: 14.975 },
};

const handleProvinceChange = (code: string) => {
  const t = CANADA_TAX[code];
  setState((s) => ({
    ...s,
    province: code,
    taxRate: t ? String(t.rate) : "13",
    taxLabel: t ? t.label : "HST",
  }));
};
```

When the user selects Ontario, `taxRate` becomes `"13"` and `taxLabel` becomes `"HST"`. Alberta gives `"5"` and `"GST"`. Quebec gives `"14.975"` and `"GST+QST"`.

---

## India: CGST+SGST Split

For intra-state GST in India, the total GST is split equally between CGST and SGST. An 18% GST invoice shows CGST 9% + SGST 9% as separate lines — not a single 18% line.

In state:

```typescript
indiaTaxType: "CGST+SGST" | "IGST"
```

In the PDF generation:

```typescript
if (state.region === "India" && state.indiaTaxType === "CGST+SGST") {
  const half = taxAmount / 2;
  drawTotal(`CGST (${parseFloat(state.taxRate) / 2}%)`, fmt(half));
  drawTotal(`SGST (${parseFloat(state.taxRate) / 2}%)`, fmt(half));
} else if (taxAmount > 0) {
  drawTotal(`${label} (${state.taxRate}%)`, fmt(taxAmount));
}
```

---

## EU Reverse Charge

For B2B cross-border EU transactions, the buyer accounts for VAT. The seller's invoice must show €0 VAT with a statutory note.

```typescript
const taxAmount = (state.region === "EU" && state.euReverseCharge)
  ? 0
  : subtotal * taxRateNum / 100;
```

In the PDF:

```typescript
if (state.region === "EU" && state.euReverseCharge) {
  page.drawText("* VAT reverse charge applies", { x: 40, y, size: 8, font, color: rgb(0.5, 0.5, 0.5) });
}
```

---

## Client-Side PDF with pdf-lib

All PDF generation happens in the browser using `pdf-lib`. No server round-trip.

```typescript
async function generateInvoicePDF(
  state: InvoiceState,
  subtotal: number,
  taxAmount: number,
  total: number
): Promise<void> {
  const pdfDoc = await PDFDocument.create();
  const page = pdfDoc.addPage([595, 842]); // A4
  const { width, height } = page.getSize();

  const fontBold = await pdfDoc.embedFont(StandardFonts.HelveticaBold);
  const font = await pdfDoc.embedFont(StandardFonts.Helvetica);

  // Professional template: indigo header band
  if (state.template === "professional") {
    page.drawRectangle({
      x: 0, y: height - 90, width, height: 90,
      color: rgb(0.286, 0.302, 0.671),
    });
  }

  // ... draw invoice content with manual x/y positioning ...

  const pdfBytes = await pdfDoc.save();
  // TypeScript fix: Uint8Array<ArrayBufferLike> → Uint8Array<ArrayBuffer>
  const blob = new Blob([new Uint8Array(pdfBytes)], { type: "application/pdf" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `${state.invoiceType.replace(" ", "-")}-${state.invoiceNumber}.pdf`;
  a.click();
  URL.revokeObjectURL(url);
}
```

> **TypeScript note:** `pdfDoc.save()` returns `Uint8Array<ArrayBufferLike>`, but `new Blob()` expects `BlobPart` which requires `Uint8Array<ArrayBuffer>`. Wrapping with `new Uint8Array(pdfBytes)` resolves the type mismatch.

---

## Auto-Increment Invoice Numbers

Each new invoice gets the next number in sequence, persisted across sessions:

```typescript
function getNextInvoiceNumber(): string {
  if (typeof window === "undefined") return "INV-001";
  const last = localStorage.getItem("ult-invoice-last-number");
  const n = last ? parseInt(last) + 1 : 1;
  localStorage.setItem("ult-invoice-last-number", String(n));
  return `INV-${String(n).padStart(3, "0")}`;
}
```

---

## localStorage Persistence

Users shouldn't have to re-enter their business name and address every time. We persist everything except the invoice number and dates (those are always fresh):

```typescript
type PersistedInvoiceFields = Pick<InvoiceState,
  "region" | "template" | "senderName" | "senderAddress" | "senderTaxId" | "senderEmail" |
  "clientName" | "clientAddress" | "clientTaxId" | "province" | "taxRate" | "taxLabel" |
  "indiaTaxType" | "euReverseCharge" | "items" | "notes" | "terms" | "upiId" |
  "currency" | "customTaxLabel"
>;

// Save with 500ms debounce on every state change
useEffect(() => {
  const timer = setTimeout(() => saveInvoiceState(state), 500);
  return () => clearTimeout(timer);
}, [state]);

// On mount: restore saved state, apply fresh invoice number + dates
useEffect(() => {
  const saved = loadSavedInvoiceState();
  const invoiceNum = getNextInvoiceNumber();

  if (saved?.region) {
    applyRegion(saved.region); // user's saved preference wins
  } else {
    applyRegion("India"); // default while IP detection runs
    detectInvoiceRegion().then(applyRegion);
  }
}, []);
```

---

## IP-Based Region Detection

`navigator.language` is unreliable — `en-US` is the default on most Windows machines regardless of the user's actual location. We use an IP geolocation API instead:

```typescript
async function detectInvoiceRegion(): Promise<Region> {
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 2000);
    const res = await fetch("https://api.country.is/", { signal: controller.signal });
    clearTimeout(timer);
    const data = await res.json() as { country: string };
    return countryToInvoiceRegion(data.country);
  } catch {
    return langFallback(); // navigator.language as last resort
  }
}
```

2-second timeout via `AbortController`. If the API is unreachable, falls back to language detection. If the user already has a saved region preference in localStorage, we skip the API call entirely.

---

## Live Preview

The right panel renders a live HTML invoice preview that mirrors the PDF layout. It updates as the user types — no delay, no "preview" button. When they download, the PDF matches what they see.

The key is keeping the preview HTML and the PDF layout in sync — same field references, same conditional rendering logic (India shows HSN/SAC columns, EU shows reverse charge note, Canada shows province label).

---

## Result

The tool is live at [ultimatetools.io/tools/misc-tools/invoice-generator/](https://ultimatetools.io/tools/misc-tools/invoice-generator/). No account, no server upload, no watermark.

Key decisions:
- **pdf-lib over jsPDF** — better TypeScript types, more control over layout
- **localStorage over server** — zero cost, works offline, private by default
- **IP detection over navigator.language** — accurate for the primary Indian market where en-US is common
- **Region config object** — adding an 8th region is a one-line addition to `REGION_CONFIG`