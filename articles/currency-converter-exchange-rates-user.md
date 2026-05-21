# Free Currency Converter Online — Live ECB Rates, 30+ Currencies, Works Offline

Converting currencies sounds simple — until you need to know which rate to trust, whether your bank is charging a fair margin, or why a tool still works when you're offline at 30,000 feet. Here's how ECB exchange rates work, how the converter is built, and which 32 currencies it supports.

## Where the Rates Come From — The ECB

The exchange rates in this tool come from the **European Central Bank (ECB)**, fetched via the [Frankfurter API](https://api.frankfurter.dev). The ECB publishes reference exchange rates once per business day, usually around 16:00 CET. These are the **mid-market rates** — the midpoint between buy and sell prices on the interbank market.

Why ECB rates?
- **Free** — no API key, no account, no rate limits
- **Trustworthy** — same rates used by Bloomberg, Reuters, and central banks worldwide
- **Consistent** — published once per day, not subject to real-time speculative swings
- **Open data** — public domain, no terms of service restrictions on use

The mid-market rate is the fairest reference rate. When your bank gives you a worse rate, the difference is their margin — typically 1–4% for retail customers, up to 8% at airport kiosks.

## How the 12-Hour Cache Works

Fetching rates on every page load would hit the API unnecessarily and waste bandwidth. The converter uses a **localStorage cache with a 12-hour TTL**:

```javascript
const CACHE_KEY = "cc-rates-v1";
const CACHE_TTL = 12 * 60 * 60 * 1000; // 12 hours in ms

async function fetchRates() {
  // Check cache first
  const raw = localStorage.getItem(CACHE_KEY);
  if (raw) {
    const { rates, fetchedAt } = JSON.parse(raw);
    if (Date.now() - fetchedAt < CACHE_TTL) {
      return rates; // Use cached — no API call
    }
  }

  // Cache stale or missing — fetch fresh
  const res = await fetch("https://api.frankfurter.dev/v1/latest?from=USD");
  const data = await res.json();
  const freshRates = { USD: 1, AED: 3.6725, ...data.rates };

  localStorage.setItem(CACHE_KEY, JSON.stringify({
    rates: freshRates,
    fetchedAt: Date.now(),
  }));

  return freshRates;
}
```

**Result:** 1 API call per browser per 12 hours, regardless of how many conversions the user does. The ECB only updates once per day anyway, so a 12-hour window gives you the fresh rate within half a day of ECB publishing it.

## Why AED Is Hardcoded

The UAE Dirham (AED) is **officially pegged to the US Dollar** at a fixed rate of **3.6725 AED = 1 USD** by the UAE Central Bank. This rate has been fixed since 1997 and does not fluctuate. The ECB doesn't publish AED data because a pegged currency has no meaningful mid-market rate — it's always 3.6725.

So rather than fetch it (it doesn't exist in ECB data) or omit it (AED is heavily used in business), it's hardcoded:

```javascript
const freshRates = { USD: 1, AED: 3.6725, ...data.rates };
```

## Offline Fallback Rates

If the API is unreachable (no internet, API outage), the tool falls back to a hardcoded set of approximate rates:

```javascript
const FALLBACK_RATES = {
  USD: 1, EUR: 0.92, GBP: 0.79, CAD: 1.36, AUD: 1.53,
  INR: 83.5, JPY: 149.5, CHF: 0.90, SGD: 1.34, AED: 3.6725,
  // ... 22 more currencies
};
```

These are "last known good" approximations — accurate enough for rough estimates but not for financial transactions. The UI shows "Approximate rates — API unavailable" when falling back, so users know the rates aren't live.

## All 32 Supported Currencies

| Code | Currency | Code | Currency |
|---|---|---|---|
| USD | US Dollar | NOK | Norwegian Krone |
| EUR | Euro | SEK | Swedish Krona |
| GBP | British Pound | DKK | Danish Krone |
| CAD | Canadian Dollar | PLN | Polish Złoty |
| AUD | Australian Dollar | TRY | Turkish Lira |
| INR | Indian Rupee | KRW | South Korean Won |
| JPY | Japanese Yen | CNY | Chinese Yuan |
| CHF | Swiss Franc | MYR | Malaysian Ringgit |
| SGD | Singapore Dollar | THB | Thai Baht |
| AED | UAE Dirham* | IDR | Indonesian Rupiah |
| NZD | New Zealand Dollar | PHP | Philippine Peso |
| HKD | Hong Kong Dollar | ILS | Israeli Shekel |
| MXN | Mexican Peso | CZK | Czech Koruna |
| BRL | Brazilian Real | HUF | Hungarian Forint |
| ZAR | South African Rand | RON | Romanian Leu |
| ISK | Icelandic Króna | BGN | Bulgarian Lev |

*AED pegged to USD at 3.6725 — hardcoded, not fetched from ECB.

## Conversion Formula

All ECB rates are expressed relative to USD (base = 1 USD). The conversion between any two currencies is:

```
result = amount × (toRate / fromRate)
```

Example: Convert 100 EUR to GBP
- EUR rate (relative to USD): 0.92 (1 USD = 0.92 EUR, so 1 EUR = 1/0.92 USD)
- GBP rate (relative to USD): 0.79

```
result = 100 × (0.79 / 0.92) = 85.87 GBP
```

No separate EUR→GBP rate needed — all conversions go through USD as the common base.

## When to Use It

**Before sending an invoice in a foreign currency** — confirm the rate before quoting a price in GBP, EUR, or CAD. Then use the [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) to create the invoice with the right currency.

**For travel budget planning** — convert your home currency to the destination currency before you go. The offline cache means it works on the plane too.

**For international online shopping** — a product listed in USD on a US site might be cheaper (or more expensive) than the equivalent in EUR on a European site once you account for the exchange rate.

**For freelancers billing internationally** — know what 1,000 USD means in your local currency before accepting a project rate.

## Related Tools

- [Currency Converter](https://ultimatetools.io/tools/misc-tools/currency-converter/) — live ECB rates, 32 currencies, 12h cache, works offline
- [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) — create professional invoices in any currency for Canada, UK, Australia, EU & USA
- [Unit Converter](https://ultimatetools.io/tools/misc-tools/unit-converter/) — convert length, weight, temperature, speed and more
- [EMI Calculator](https://ultimatetools.io/tools/misc-tools/emi-calculator/) — calculate loan EMI with amortization table

---

Convert between 32 currencies using live ECB rates — free, no signup, works offline: [Currency Converter](https://ultimatetools.io/tools/misc-tools/currency-converter/)